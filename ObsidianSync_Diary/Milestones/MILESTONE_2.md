# ObsidianSync — Report Milestone 2

## Obiettivi Raggiunti

- Implementata architettura event-driven completa: `SyncEvent`, `EventType`, `SyncEventQueue`
- Implementato `SyncOrchestrator` con worker loop, retry con exponential backoff, shutdown pulito
- Esteso `GitService` con `commitLocal()`, `hasChanges()`, `hasUncommittedChanges()`, `stash()`, `stashPop()`
- Implementato `NotificationHook` come interfaccia + `LogNotificationHook` come default
- Implementato `AutosaveScheduler` con `ScheduledExecutorService` e gestione shutdown ordinata
- Aggiornato `Main` con dependency injection manuale e dispatch CLI → evento
- Adottato Git Flow come branching strategy — feature branch per ogni obiettivo del grooming
- Riorganizzata struttura package: `orchestrator/`, `scheduler/`, `service/`, `hook/`
- Allineato target compilazione a Java 21 LTS
- Aggiornato DTR con le decisioni architetturali della milestone

---

## Problematiche Affrontate

| Problema | Risoluzione |
|---|---|
| `Process` non `AutoCloseable` su Java 21/25 | Rimosso try-with-resources su `Process`, mantenuto solo su `BufferedReader` |
| `ExecutorService` nel costruttore di `SyncOrchestrator` senza utilizzo reale | Rimosso — sostituito con `Thread` grezzo, più semplice e semanticamente corretto |
| `notify()` in conflitto con `Object.notify()` | Rinominato `onFailure(event, message)` con aggiunta del parametro `message` |
| `stash pop` su stash vuoto generava exit code 1 | Introdotto guard `hasUncommittedChanges()` prima di `stash()` e `stashPop()` |
| `git diff --quiet` insufficiente per rilevare modifiche staged | Usato `git status --porcelain` — output stabile, locale-indipendente, copre tutti i casi |
| Task `scheduleAtFixedRate` che si interrompe silenziosamente su eccezione | Wrappato il task in try/catch — comportamento noto di `ScheduledExecutorService` |
| `new Thread()` passato come `Runnable` a `scheduleAtFixedRate` | Sostituito con lambda — semanticamente corretto |
| `ScheduledFuture` non salvato | Salvato come campo — necessario per cancellazione selettiva futura |
| Ordine shutdown scheduler/orchestratore | Scheduler fermato prima — evita publish su coda non più consumata |

---

## Cheat-Sheet Acquisiti

### Java Concurrency
- `Executors.newSingleThreadScheduledExecutor()` — factory per executor schedulato a thread singolo
- `scheduleAtFixedRate(task, delay, period, unit)` — clock fisso: l'intervallo parte dall'inizio del task
- `scheduleWithFixedDelay(task, delay, period, unit)` — catena: l'intervallo parte dalla fine del task
- `scheduleAtFixedRate` interrompe silenziosamente la schedulazione su eccezione unchecked — wrappare sempre in try/catch
- `ScheduledFuture.cancel(false)` — cancella il task senza interrompere l'esecuzione in corso
- `executor.awaitTermination(n, unit)` — attende la terminazione con timeout; se scade → `shutdownNow()`
- `Thread.currentThread().isInterrupted()` — check del flag di interruzione nel worker loop
- `PriorityBlockingQueue.take()` — bloccante, lancia `InterruptedException` su interrupt
- `synchronized` su sequenze composte (check + remove + insert) — atomicità equivalente a transazione DB

### Java General
- `Runnable` come lambda: `() -> { ... }` invece di `new Thread() { @Override public void run() {...} }`
- `Process` implementa `AutoCloseable` solo da Java 26 — su Java 21 non usarlo in try-with-resources
- `maven.compiler.source/target` — compilazione a livello inferiore con JDK superiore installato

### Git
- `git status --porcelain` — output stabile e locale-indipendente per uso programmatico
- `git status --porcelain` output vuoto = working tree pulita; non vuoto = modifiche presenti
- `git diff --quiet` — controlla solo modifiche non staged; insufficiente come guard generale

### Design Pattern
- **Dependency Injection manuale** — `Main` come unico punto di wiring; nessuna classe istanzia direttamente un'altra
- **Dependency Inversion** — `SyncOrchestrator` dipende da `NotificationHook` (interfaccia), non da `LogNotificationHook` (implementazione)
- **Publisher/Subscriber** — `AutosaveScheduler` pubblica, `SyncOrchestrator` consuma; nessun accoppiamento diretto
- **Command Pattern** — `SyncEvent` come comando eseguibile con tipo, timestamp e stato di retry

---

## Valutazione Personal Trainer

### Punteggio Complessivo: 8.6 / 10

| Dimensione | Voto | Note |
|---|---|---|
| Velocità di acquisizione know-how | 9/10 | Concurrency Java assorbita rapidamente e applicata correttamente. `PriorityBlockingQueue`, `ScheduledExecutorService`, `synchronized`, shutdown hook — tutti compresi e usati senza copia-incolla. |
| Velocità di delivery | 8/10 | Ritmo sostenuto nonostante la complessità crescente. Qualche rallentamento fisiologico su concetti nuovi (ExecutorService, thread lifecycle), risolto autonomamente. |
| Impegno | 9/10 | Ha cercato attivamente le risposte (Stack Overflow su scheduleAtFixedRate) prima di chiedere. Ha ragionato sull'ordine di shutdown in modo indipendente e corretto. |
| Qualità del codice | 8/10 | Struttura pulita, separazione delle responsabilità rispettata. Punti di miglioramento: task senza try/catch, `ScheduledFuture` non salvato — entrambi rilevati in fase di review e corretti. |

### Note del Trainer

La milestone 2 è stata significativamente più ambiziosa della prima — architettura event-driven, concorrenza, pattern di design, branching strategy. Il candidato ha mantenuto il filo senza perdersi nella complessità, dimostrando capacità di astrazione crescente.

Il salto qualitativo più evidente è nella **autonomia decisionale**: l'ordine di shutdown, la scelta tra `scheduleAtFixedRate` e `scheduleWithFixedDelay`, la motivazione del guard `hasUncommittedChanges()` — tutti ragionamenti condotti in modo indipendente e corretti.

**Aree su cui lavorare nella Milestone 3:**
- Introdurre i primi unit test — `LogService`, `SyncEventQueue` e `GitService` sono testabili in isolamento
- Validazione delle properties in ingresso prima dell'istanziazione dei servizi
- Considerare la gestione di `NetworkException` vs `GitException` per differenziare retry e no-retry
- Multi-vault support: `config.properties` con N vault profile — obiettivo naturale della milestone successiva
