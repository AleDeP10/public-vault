# Java Collections Avanzate — Generics, Covarianza, Casting

## 1. Perché `List<SystemPattern>` non è un `List<GitignorePattern>`

`SystemPattern` estende `GitignorePattern`. Intuitivamente sembrerebbe che una lista di `SystemPattern` sia anche una lista di `GitignorePattern`. In Java **non funziona così**.

```java
List<SystemPattern> systems = new ArrayList<>();
List<GitignorePattern> patterns = systems;   // ERRORE di compilazione
```

Il motivo è la **sicurezza dei tipi**. Se la seconda riga compilasse, potresti fare:

```java
patterns.add(new GitignorePattern(...));   // aggiunge un NON-SystemPattern
SystemPattern sp = systems.get(0);        // ClassCastException a runtime
```

Java blocca questo a compile-time. I generici in Java sono **invarianti** per default.

---

## 2. Le tre varianze

|Varianza|Sintassi|Può leggere|Può scrivere|
|---|---|---|---|
|Invariante|`List<T>`|✅|✅|
|Covariante|`List<? extends T>`|✅|❌|
|Controvariante|`List<? super T>`|❌ (solo Object)|✅|

```java
// COVARIANTE — puoi leggere T, non puoi aggiungere
List<? extends GitignorePattern> lista = new ArrayList<SystemPattern>();
GitignorePattern p = lista.get(0);   // ok
lista.add(new SystemPattern(...));   // ERRORE — il compilatore non sa il tipo esatto

// CONTROVARIANTE — puoi aggiungere T, leggere solo come Object
List<? super SystemPattern> lista = new ArrayList<GitignorePattern>();
lista.add(new SystemPattern(...));   // ok
Object o = lista.get(0);            // solo Object
```

**Regola mnemonica (PECS)**: Producer Extends, Consumer Super.

- Se la lista **produce** elementi che usi → `extends`
- Se la lista **consuma** elementi che ci metti → `super`

---

## 3. Cast sicuro di una lista — il pattern con Stream

Quando sai che tutti gli elementi in una `List<GitignorePattern>` sono in realtà `SystemPattern`, hai due opzioni.

**Opzione A — cast grezzo (da evitare)**

```java
@SuppressWarnings("unchecked")
List<SystemPattern> system = (List<SystemPattern>) (List<?>) systemRaw;
```

Compila, ma è un cast cieco. Se un elemento non è `SystemPattern`, esplode a runtime con `ClassCastException` nel momento in cui lo usi.

**Opzione B — cast via Stream (raccomandato)**

```java
List<SystemPattern> system = systemRaw.stream()
    .filter(SystemPattern.class::isInstance)   // tieni solo i SystemPattern
    .map(SystemPattern.class::cast)            // cast type-safe elemento per elemento
    .collect(Collectors.toList());
```

`SystemPattern.class::isInstance` è equivalente a `p -> p instanceof SystemPattern`. `SystemPattern.class::cast` è equivalente a `p -> (SystemPattern) p`.

La differenza: il `filter` garantisce che il `cast` non fallisca mai. Se un elemento non è `SystemPattern` viene silenziosamente scartato invece di lanciare eccezione.

---

## 4. `Collectors.groupingBy` — partizionare per chiave

```java
Map<PatternLevel, List<GitignorePattern>> partitioned = patterns.stream()
    .collect(Collectors.groupingBy(GitignorePattern::getLevel));
```

Produce una `Map` dove ogni chiave è un valore di `PatternLevel` e il valore è la lista di pattern con quel livello. Se nessun elemento ha un dato livello, quella chiave semplicemente non esiste nella mappa.

**Attenzione al NPE**: `partitioned.get(PatternLevel.USER)` restituisce `null` se non ci sono pattern USER. Usa sempre `getOrDefault`:

```java
List<GitignorePattern> user = partitioned.getOrDefault(PatternLevel.USER, List.of());
```

---

## 5. `Collectors.toMap` con merge function

```java
Map<String, Boolean> appNegations = appRaw.stream()
    .collect(Collectors.toMap(
        GitignorePattern::getPattern,   // chiave
        GitignorePattern::isNegated,    // valore
        (existing, replacement) -> replacement   // merge: in caso di chiave duplicata, prendi l'ultimo
    ));
```

Senza la merge function, se due elementi hanno lo stesso pattern, `toMap` lancia `IllegalStateException`. La merge function decide cosa fare: qui prende il valore più recente.

---

## 6. `peek` — effetti collaterali nello stream senza consumarlo

```java
List<SystemPattern> system = systemRaw.stream()
    .filter(SystemPattern.class::isInstance)
    .map(SystemPattern.class::cast)
    .peek(p -> {
        if (p.isNegated()) logService.warn("Cannot negate SYSTEM: " + p.getPattern());
    })
    .collect(Collectors.toList());
```

`peek` esegue un'azione su ogni elemento **senza modificarlo e senza consumare lo stream**. È pensato per logging e debugging. Non usarlo per mutare lo stato degli elementi — per quello usa `forEach` sul risultato finale.

---

## 7. `List.of()` vs `new ArrayList<>()` — immutabilità

```java
List<String> immutable = List.of("a", "b");   // Java 9+, non modificabile
List<String> mutable   = new ArrayList<>(List.of("a", "b"));   // copia modificabile

immutable.add("c");   // UnsupportedOperationException
mutable.add("c");     // ok
```

`stream().toList()` in Java 16+ restituisce una lista **non modificabile**. Se serve modificarla:

```java
List<GitignorePattern> mutable = new ArrayList<>(stream.toList());
```

---

## 8. Copia difensiva — quando e perché

Un metodo che restituisce una collection interna espone lo stato interno alla mutazione dall'esterno.

```java
// PERICOLOSO — il chiamante può modificare la lista interna
public List<Vault> findAll() {
    return vaults;   // reference diretta
}

// SICURO — copia difensiva
public List<Vault> findAll() {
    return new ArrayList<>(vaults);   // nuova lista, stessi elementi
}
```

Attenzione: la copia difensiva protegge la **struttura** (add/remove), non gli **elementi**. Se `Vault` è mutabile, il chiamante può comunque modificare i singoli oggetti.

---

## 9. `stream()` vs `parallelStream()`

Per le liste di NomadSync (decine di elementi al massimo) usa sempre `stream()`. `parallelStream()` ha overhead di thread management che conviene solo sopra le migliaia di elementi e solo per operazioni CPU-bound senza side effects.

---

## 10. Riepilogo rapido dei Collectors più usati

```java
// lista (non modificabile da Java 16)
.collect(Collectors.toList())    // modificabile, pre-16
.toList()                        // non modificabile, Java 16+

// set
.collect(Collectors.toSet())

// mappa
.collect(Collectors.toMap(keyFn, valueFn))
.collect(Collectors.toMap(keyFn, valueFn, mergeFn))

// partizionamento per chiave
.collect(Collectors.groupingBy(classifier))

// stringa
.collect(Collectors.joining(", "))
.collect(Collectors.joining(", ", "[", "]"))

// conteggio
.collect(Collectors.counting())
```