# Java File System — Path, Files, Walkers, Visitors

## 1. `Path` — rappresentare un percorso

`Path` è un'interfaccia, non una classe. Rappresenta un percorso nel filesystem **senza aprire nulla**. Non tocca il disco finché non chiami qualcosa che lo richiede esplicitamente.

```java
Path p = Path.of("C:\\Users\\aless\\vault");   // Windows
Path p = Path.of("/home/aless/vault");          // Unix

// costruzione per composizione — preferibile a concatenazione di stringhe
Path gitignore = vaultPath.resolve(".gitignore");   // vaultPath + ".gitignore"
Path parent    = vaultPath.getParent();             // un livello su
Path name      = vaultPath.getFileName();           // solo l'ultimo segmento
Path relative  = vaultPath.relativize(subPath);     // percorso relativo tra due assoluti
Path absolute  = relative.toAbsolutePath();         // da relativo ad assoluto

// conversione con il vecchio API
File f = p.toFile();   // Path → File (legacy)
Path p = f.toPath();   // File → Path (legacy)
```

---

## 2. `Files` — operazioni sul filesystem

`Files` è una classe di utility con tutti i metodi statici per operare su `Path`. Non istanziate mai `File` se potete usare `Files`.

```java
// esistenza e tipo
Files.exists(path)
Files.isDirectory(path)
Files.isRegularFile(path)

// lettura
String content   = Files.readString(path);
List<String> lines = Files.readAllLines(path);
byte[] bytes     = Files.readAllBytes(path);

// scrittura (sovrascrive)
Files.writeString(path, "contenuto");
Files.write(path, "contenuto".getBytes());
Files.write(path, List.of("riga1", "riga2"));

// scrittura in append mode
Files.writeString(path, "aggiunta", StandardOpenOption.APPEND);

// copia
Files.copy(source, target);
Files.copy(source, target, StandardCopyOption.REPLACE_EXISTING);

// spostamento/rinomina
Files.move(source, target);

// cancellazione
Files.delete(path);                          // lancia eccezione se non esiste
Files.deleteIfExists(path);                  // no-op se non esiste

// creazione directory
Files.createDirectory(path);                 // solo un livello
Files.createDirectories(path);               // crea tutto l'albero (equivalente mkdir -p)

// listare una directory (non ricorsivo)
try (Stream<Path> stream = Files.list(path)) {
    stream.filter(Files::isRegularFile).forEach(System.out::println);
}
// IMPORTANTE: Files.list() restituisce uno Stream che deve essere chiuso
// per questo si usa try-with-resources
```

---

## 3. Cancellazione ricorsiva — il pattern standard

Non esiste un `Files.deleteRecursively()`. Il pattern idiomatico in Java usa `Files.walk()`:

```java
private void deleteRecursively(Path root) throws IOException {
    Files.walk(root)
         .sorted(Comparator.reverseOrder())   // file PRIMA delle loro directory
         .map(Path::toFile)                   // Path → File per accedere a delete()
         .forEach(File::delete);
}
```

**Perché `reverseOrder()`?** `Files.walk()` visita prima le directory poi i contenuti. Se cancelli la directory prima dei file dentro, il sistema operativo rifiuta perché non è vuota. Invertendo l'ordine si garantisce che i file vengano cancellati prima delle loro cartelle.

---

## 4. `Files.walkFileTree` e `FileVisitor` — visita con controllo

`Files.walkFileTree` è la versione avanzata di `Files.walk`. Ti dà controllo completo su ogni nodo dell'albero tramite callback.

```java
Files.walkFileTree(startPath, new FileVisitor<Path>() {

    @Override
    public FileVisitResult preVisitDirectory(Path dir, BasicFileAttributes attrs)
            throws IOException {
        // chiamato PRIMA di entrare in una directory
        // puoi decidere se entrarci o saltarla
        return FileVisitResult.CONTINUE;
    }

    @Override
    public FileVisitResult visitFile(Path file, BasicFileAttributes attrs)
            throws IOException {
        // chiamato per ogni file
        return FileVisitResult.CONTINUE;
    }

    @Override
    public FileVisitResult visitFileFailed(Path file, IOException exc)
            throws IOException {
        // chiamato se un file non è accessibile (permessi, ecc.)
        return FileVisitResult.CONTINUE;
    }

    @Override
    public FileVisitResult postVisitDirectory(Path dir, IOException exc)
            throws IOException {
        // chiamato DOPO aver visitato tutti i contenuti di una directory
        return FileVisitResult.CONTINUE;
    }
});
```

### FileVisitResult — le quattro opzioni

|Valore|Effetto|
|---|---|
|`CONTINUE`|prosegui normalmente|
|`SKIP_SUBTREE`|non entrare in questa directory (solo da `preVisitDirectory`)|
|`SKIP_SIBLINGS`|salta gli altri elementi allo stesso livello|
|`TERMINATE`|interrompi l'intera visita|

---

## 5. `SimpleFileVisitor` — implementazione parziale

Implementare `FileVisitor` obbliga a scrivere tutti e quattro i metodi anche se ne usi solo due. `SimpleFileVisitor` è una classe astratta che li implementa tutti con comportamento di default (`CONTINUE`), permettendoti di fare override solo dei metodi che ti interessano.

```java
Files.walkFileTree(vaultDir, new SimpleFileVisitor<Path>() {

    @Override
    public FileVisitResult preVisitDirectory(Path dir, BasicFileAttributes attrs)
            throws IOException {
        Path relative = vaultDir.relativize(dir);
        if (matchers.stream().anyMatch(m -> m.matches(relative))) {
            return FileVisitResult.SKIP_SUBTREE;   // directory ignorata — non entrare
        }
        Files.createDirectories(snapshotDir.resolve(relative));
        return FileVisitResult.CONTINUE;
    }

    @Override
    public FileVisitResult visitFile(Path file, BasicFileAttributes attrs)
            throws IOException {
        Path relative = vaultDir.relativize(file);
        if (matchers.stream().noneMatch(m -> m.matches(relative))) {
            Files.copy(file, snapshotDir.resolve(relative));
        }
        return FileVisitResult.CONTINUE;
    }
});
```

**Cosa succede qui passo per passo:**

1. `walkFileTree` parte da `vaultDir` e visita ogni nodo dell'albero.
2. Per ogni **directory**, chiama `preVisitDirectory` prima di entrarci.
    - Se il path relativo corrisponde a un pattern gitignore → `SKIP_SUBTREE`: l'intera cartella viene saltata, nessun file dentro verrà visitato.
    - Altrimenti → crea la directory corrispondente nello snapshot e prosegui.
3. Per ogni **file**, chiama `visitFile`.
    - Se il path relativo NON corrisponde a nessun matcher → copialo nello snapshot.
    - Se corrisponde → ignoralo (non copiare).

### Il ruolo di `relativize`

```java
Path relative = vaultDir.relativize(file);
```

`relativize` calcola il percorso relativo di `file` rispetto a `vaultDir`. Serve perché i `PathMatcher` compilati da `.gitignore` lavorano su path relativi, non assoluti.

Esempio:

```
vaultDir = C:\vault\notes
file     = C:\vault\notes\.obsidian\cache\index.json
relative = .obsidian\cache\index.json        ← questo matcha il pattern
```

---

## 6. `PathMatcher` — compilare e usare pattern glob

```java
// compilazione
PathMatcher matcher = FileSystems.getDefault().getPathMatcher("glob:.obsidian/cache");
PathMatcher matcher = FileSystems.getDefault().getPathMatcher("glob:*.tmp");
PathMatcher matcher = FileSystems.getDefault().getPathMatcher("glob:**/.git");

// utilizzo
boolean matches = matcher.matches(Path.of(".obsidian/cache"));   // true

// in NomadSync — compilazione da lista di pattern
List<PathMatcher> matchers = patterns.stream()
    .filter(p -> !p.isNegated())
    .map(p -> FileSystems.getDefault().getPathMatcher("glob:" + p.getPattern()))
    .toList();

// verifica
boolean excluded = matchers.stream().anyMatch(m -> m.matches(relative));
```

### Sintassi glob

|Pattern|Significato|
|---|---|
|`*.txt`|qualsiasi file con estensione `.txt`|
|`**/.git`|`.git` a qualsiasi profondità|
|`?`|un singolo carattere qualsiasi|
|`{a,b,c}`|una delle alternative|
|`.obsidian/plugins/*/data.json`|`*` = un segmento qualsiasi|

---

## 7. `try-with-resources` con stream di file

`Files.list()`, `Files.walk()`, `Files.lines()` restituiscono `Stream<Path>` che wrappano risorse OS (file descriptor). Devono essere chiusi.

```java
// CORRETTO — try-with-resources chiude lo stream automaticamente
try (Stream<Path> stream = Files.list(backupsRoot)) {
    List<Path> snapshots = stream
        .filter(p -> p.getFileName().toString().startsWith(vaultName + "_"))
        .sorted()
        .toList();
}

// SBAGLIATO — stream non chiuso, file descriptor leak
List<Path> snapshots = Files.list(backupsRoot)
    .filter(...)
    .toList();   // il file descriptor rimane aperto
```

---

## 8. `BasicFileAttributes` — metadati del file

Disponibile nei callback di `FileVisitor` senza costo aggiuntivo (il walker li legge comunque):

```java
@Override
public FileVisitResult visitFile(Path file, BasicFileAttributes attrs) {
    long size        = attrs.size();
    boolean isDir    = attrs.isDirectory();
    boolean isFile   = attrs.isRegularFile();
    FileTime created  = attrs.creationTime();
    FileTime modified = attrs.lastModifiedTime();
    return FileVisitResult.CONTINUE;
}
```

---

## 9. Riepilogo — quando usare cosa

| Caso d'uso                            | API consigliata                                       |
| ------------------------------------- | ----------------------------------------------------- |
| Leggere / scrivere un file            | `Files.readString` / `Files.writeString`              |
| Creare directory (incluse intermedie) | `Files.createDirectories`                             |
| Listare una directory (non ricorsiva) | `Files.list()` in try-with-resources                  |
| Cancellazione ricorsiva               | `Files.walk()` + `sorted(reverseOrder())`             |
| Copia/backup con esclusioni           | `Files.walkFileTree` + `SimpleFileVisitor`            |
| Matching di pattern gitignore         | `FileSystems.getDefault().getPathMatcher("glob:...")` |
| Percorso relativo tra due assoluti    | `parent.relativize(child)`                            |
| Costruire path in modo sicuro         | `parent.resolve("subfolder")`                         |