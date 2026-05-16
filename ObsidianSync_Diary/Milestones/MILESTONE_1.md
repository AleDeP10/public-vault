# ObsidianSync — Report Milestone 1

## Obiettivi Raggiunti

- Definita l'architettura di sincronizzazione: Git-based, innescata dal logon/logoff Windows tramite Task Scheduler
- Definita la struttura del progetto: Maven, fat JAR, risorse copiate a fianco dell'artefatto
- Implementato `Main.java`: parsing argomenti CLI, caricamento properties, dispatch delle operazioni
- Implementato `GitService.java`: `push`, `pull`, `autosave` tramite `ProcessBuilder`
- Implementato `LogService.java`: logging a livelli, append-only, thread-safe tramite `synchronized`
- Configurato `config.dev.properties` e `.gitignore` con esclusione delle credenziali
- Creati `ObsidianSync.bat` (generico) e `ObsidianSyncPush.bat` (shortcut specializzato)
- Risolti autonomamente multipli conflitti Git e problemi di configurazione del repository
- Scaffolding di `AutosaveScheduler` e `SyncOrchestrator` — implementazione rimandata alla Milestone 2

---

## Problematiche Affrontate

| Problema                                         | Risoluzione                                                                                                     |
| ------------------------------------------------ | --------------------------------------------------------------------------------------------------------------- |
| GitHub 2FA bloccata sul vecchio telefono         | Identificata la causa radice (segreto TOTP legato al vecchio device); risoluzione demandata all'accesso da casa |
| `git branch` non mostrava nulla dopo `git init`  | Compreso che i branch senza commit non vengono listati; proceduto con il primo commit                           |
| Errori ripetuti `unrelated histories`            | Applicato il flag `--allow-unrelated-histories` in modo consistente                                             |
| `.idea/workspace.xml` bloccava il merge          | Usato `git rm --cached -r .idea/` per togliere dal tracking i file IDE; aggiornato `.gitignore`                 |
| `remote rejected` sulla cancellazione del branch | Identificato il lock del branch di default su GitHub; risolto da interfaccia Settings                           |
| `merge --abort` falliva per file non staged      | Applicato `git reset --hard HEAD` per pulire l'indice                                                           |
| Finestra `.bat` che si chiudeva immediatamente   | Aggiunto `pause` dopo l'invocazione del JAR                                                                     |
| `FileWriter` sovrascriveva il file di log        | Aggiunto il flag `append=true` al costruttore di `FileWriter`                                                   |
| Catch `FileNotFoundException` irraggiungibile    | Compresa la gerarchia delle eccezioni; consolidato in un unico catch `IOException`                              |
| Try/catch duplicato nei tre case dello switch    | Refactor: try/catch unificato all'esterno dello switch                                                          |

---

## Cheat-Sheet Acquisiti

### Git
 - `git remote add origin <url>` + `git push -u origin main` — collega la repo locale al remoto e imposta il tracking branch; i push successivi saranno semplicemente `git push`
- `git rm --cached -r <path>` — rimuove dal tracking senza cancellare i file locali
- `git pull --allow-unrelated-histories` — unisce branch senza antenato comune
- `git pull -X theirs` — risolve i conflitti preferendo il remoto
- `git stash / stash pop` — mette da parte e ripristina le modifiche non committate
- `git push origin --delete <branch>` — cancella un branch remoto
- `.gitignore` agisce solo sui file non ancora tracciati — usare `git rm --cached` per i già tracciati

### Windows / Batch
- `%1`, `%2` — argomenti posizionali nel `.bat`
- `exit /b` vs `exit` — esce dallo script vs chiude la finestra del terminale
- `call` — invoca un altro `.bat` e restituisce il controllo al chiamante
- `pause` — tiene aperta la finestra in attesa di un tasto
- `AppData` è nascosta per default — abilitare da Visualizza in Explorer
- Task Scheduler: trigger su logon/logoff/schedule; azione punta a `java -jar`

### Java
- `ProcessBuilder` — lancia processi esterni; `.directory()` imposta la working directory
- `redirectErrorStream(true)` — unifica stdout e stderr in un unico stream
- `process.waitFor()` — va chiamato **dopo** la lettura dello stream per evitare deadlock
- `synchronized` su metodo d'istanza — acquisisce il lock su `this`, serializza le chiamate concorrenti
- `new FileWriter(file, true)` — apre in modalità append
- `LocalDateTime.now().format(DateTimeFormatter.ofPattern(...))` — generazione timestamp
- `"pattern".formatted(args)` — interpolazione moderna (Java 15+)
- `Thread.currentThread().interrupt()` — ripristina il flag di interruzione dopo aver catturato `InterruptedException`
- `Runtime.getRuntime().addShutdownHook(new Thread(...))` — esegue codice alla chiusura della JVM
