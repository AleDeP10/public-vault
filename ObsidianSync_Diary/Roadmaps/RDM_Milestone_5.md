### Roadmap M5 — Layer 1 aggiornato

| #   | Step                                                                                        | Stato                                                 |
| --- | ------------------------------------------------------------------------------------------- | ----------------------------------------------------- |
| 1   | `EventType` — `SYNCHRONIZE` al posto di `PULL_MANUAL`/`PUSH_MANUAL`                         | ✅                                                     |
| 2   | `GitService.synchronize()` — conflict strategy completa                                     | 🔄 struttura presente, FIFO e PathMatcher da chiudere |
| 3   | `SyncOrchestrator` — switch aggiornato                                                      | ✅                                                     |
| 4   | `VaultService` — vincolo unicità `vault.name` + `makeVaultSnapshot` completo                | 🔄 snapshot stub presente                             |
| 5   | `GitignoreService` — multi-vault, `load(Vault)`, clone difensivo                            | 🔄 singleton presente, refactoring necessario         |
| 6   | `Vault` — `gitUsername` per chiave `username/vaultName`                                     | ✅ campo presente                                      |
| 7   | `VaultService` — spostare `makeVaultSnapshot` → delegare a `GitignoreService.forSnapshot()` | ⬜                                                     |
| 8   | `mvn test` verde — nessuna regressione                                                      | ⬜                                                     |

---

### Approfondimento Step 2 — `GitService.synchronize()` — cosa manca

La struttura è presente. Tre blocchi ancora da implementare:

**Blocco A — `makeVaultSnapshot` completo** Attualmente è uno stub in `VaultService`. Deve essere completato con FIFO reale e copia ricorsiva con PathMatcher. Dipende da `GitignoreService.forSnapshot()` — quindi Step 5 è prerequisito di Step 2.

**Blocco B — estrazione conflitti dall'output**

java

```java
List<String> conflicted = oursOutput.lines()
    .filter(l -> l.startsWith("Auto-merging"))
    .map(l -> l.replace("Auto-merging ", "").trim())
    .toList();
```

Già definito — da inserire nel branch conflitto.

**Blocco C — `git show FETCH_HEAD:<file>`**

java

```java
result.add(file); // non showOutput — il chiamante vuole i nomi
```

Il fix è già noto — da applicare.

---

### Approfondimento Step 4 — `VaultService` unicità nome

Tre punti:

**`create()` e `update()`** — aggiungere guard:

java

```java
private void checkNameUniqueness(String name, String excludeId) {
    vaults.values().stream()
        .filter(v -> v.getName().equals(name))
        .filter(v -> !v.getId().equals(excludeId))
        .findFirst()
        .ifPresent(v -> { throw new VaultException("duplicated vault name: " + name); });
}
```

`excludeId` serve in `update()` — non vuoi segnalare conflitto con se stesso.

**All'avvio** — in `load()` dopo aver popolato la mappa:

java

```java
Set<String> seen = new HashSet<>();
vaults.values().forEach(v -> {
    if (!seen.add(v.getName()))
        throw new VaultException("duplicated vault name in vaults.json: " + v.getName());
});
```

[NOTA] approccio difensivo: qualora vi fossero duplicati, ci limiteremmo a warn ed ignorare il duplicato.
[NOTA] stesso approccio difensivo coi campi obbligatori, se manca username o name, warn e passa oltre.

**`makeVaultSnapshot`** — da spostare fuori da `VaultService`. `VaultService` gestisce CRUD, non filesystem operations. Il metodo appartiene a un `SnapshotService` o rimane in `GitService` come operazione pre-conflitto.

[NOTA] Introduzione di SnapshotService, a tenderà sarà previsto un sistema di disaster recovery con ripristino da backup.

---

### Approfondimento Step 5 — `GitignoreService` multi-vault

Tre refactoring necessari:

**1. `load(Vault vault)` invece di `load(Path vaultPath)`** La chiave vault è `vault.getGitUsername() + "/" + vault.getName()`. Se `gitUsername` è null (vault non ancora configurato), fallback su `vault.getId()`.

[NOTA] struttureremo la UI in modo che il vault sia sempre configurato.
[PRIMARY_GOAL] unicità globale del vault fin dal costruttore, e per ottenerlo basta inserire due campi.
[REJECTED] fallback su vault.getId().

**2. Clone difensivo di `APP_PATTERN_DEFINITIONS` per vault**

java

```java
private List<AppPatterns> cloneAppPatterns() {
    return APP_PATTERN_DEFINITIONS.stream()
        .map(ap -> new AppPatterns(ap.getName(),
            ap.getPatterns().stream()
                .map(p -> new GitignorePattern(
                    p.getPattern(), p.getLevel(),
                    p.getAppName(), p.isNegated()))
                .toList()))
        .toList();
}
```

Chiamato in `load(Vault)` — ogni vault ottiene la sua copia isolata.

**3. `systemPatterns` e `appPatterns` come `Map<vaultKey, List<...>>`**

java

```java
private final Map<String, List<SystemPattern>> systemPatterns = new TreeMap<>();
private final Map<String, List<AppPatterns>>   appPatterns    = new TreeMap<>();
private final Map<String, List<GitignorePattern>> userPatterns = new TreeMap<>();
```

`load(Vault)` popola le tre mappe per `vaultKey`. `save()` e `forSnapshot()` ricevono `Vault` e recuperano dalla mappa.