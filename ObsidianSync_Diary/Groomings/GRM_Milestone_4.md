# GRM — Milestone 4: Integrazione Windows

---

## DTR

---

### [M4] NetworkException vs GitException in GitService

**Contesto**: il retry con exponential backoff deve applicarsi solo ai fallimenti di rete, non agli errori locali Git (conflitti, stash vuoto, repository corrotto). Git restituisce exit code generici — la distinzione va ricavata analizzando lo stderr.

**Decisione**: `GitService` lancia `NetworkException` per errori di connettività e `GitException` per errori locali. `SyncOrchestrator` gestisce i due casi con catch separati. Il retry si applica solo a `NetworkException`.

**Implementazione**: whitelist di pattern noti per gli errori di rete. Qualsiasi output stderr che corrisponde a un pattern della lista innesca `NetworkException`; tutto il resto innesca `GitException`.

**Pattern riconosciuti come errori di rete**: `timeout`, `Could not resolve host`, `Connection refused`, `Failed to connect`, `Network is unreachable`.

**Rischio accettato**: su installazioni Windows con Git localizzato, i messaggi stderr potrebbero essere in italiano. Da verificare sulla macchina reale prima di finalizzare la whitelist.

**Motivazione**: semantica chiara; retry applicato solo dove ha senso. Pattern standard nei sistemi distribuiti.

---

### [M4] Processo persistente con IPC rispetto a processi effimeri per task

**Contesto**: Task Scheduler avvia una nuova JVM per ogni task — pull, push, autosave. Se ogni task istanzia il proprio `SyncOrchestrator`, la coda a priorità perde significato: i task non coesistono mai nella stessa coda e non possono essere deduplicati né ordinati.

**Decisione**: un processo persistente ospita l'orchestratore. I task fungono da client che inviano un evento via socket e terminano. La tray icon è il candidato naturale come processo host — è già persistente per natura e già prevista nell'architettura.

**Modalità di avvio del JAR**:

- `tray` → avvia tray + socket server + orchestratore
- `logon` / `logoff` / `autosave` → client socket, invia evento, termina

**Alternativa scartata**: processo effimero per task — semplice, ma la coda a priorità è priva di significato e la deduplicazione è impossibile.

---

### [M4] IPC tramite socket TCP locale

**Contesto**: i task hanno bisogno di un meccanismo per inviare eventi al processo tray persistente. Tre opzioni valutate: file di lock, named pipe Windows, socket TCP locale.

**Decisione**: socket TCP locale su `localhost:4242`. Porta configurabile via `config.properties`.

**Motivazione**: bidirezionale, portabile tra Windows e macOS, zero dipendenze da API native. Il modello concettuale del socket TCP è identico a quello delle WebSocket — chi capisce uno capisce l'altro. Le named pipe hanno API Java macchinose e non sono portabili. I file di lock richiedono polling e non sono bidirezionali.

---

### [M4] Protocollo messaggi JSON sul socket

**Contesto**: una stringa semplice sul socket non è estensibile e non porta metadati.

**Decisione**: messaggi JSON con campi `event`, `vaultId`, `timestamp`.

**Motivazione**: estensibile senza rompere i client esistenti; `vaultId` è già predisposto per il supporto multi-vault; leggibile per il debugging.

---

### [M4] Retry del client socket: exponential backoff

**Contesto**: il processo tray potrebbe non aver completato lo startup quando un task di logon viene eseguito. Il client ha bisogno di una politica di retry per gestire la finestra di avvio.

**Decisione**: stessa politica dell'orchestratore — exponential backoff 30s → 60s → 120s, massimo 3 tentativi.

**Motivazione**: 30 secondi coprono qualsiasi finestra di startup realistica su un sistema moderno. Riutilizzare le stesse costanti di backoff dell'orchestratore mantiene la politica coerente in tutto il sistema.

---

### [M4] PUSH_MANUAL e comportamento della tray icon

**Contesto**: l'utente ha bisogno di un modo visibile e immediato per pushare le modifiche senza aspettare il logoff.

**Decisione**:

- **Click sinistro** → pubblica `PUSH_MANUAL` sul vault corrente direttamente sull'orchestratore interno (stessa JVM — nessun IPC necessario)
- **Click destro** → popup con elenco vault + boolean "Salva alla selezione"
    - Selezione vault → aggiorna `current-vault.json`
    - Se "Salva alla selezione" è attivo → pubblica automaticamente `PUSH_MANUAL` sul vault appena selezionato

**Motivazione**: "Salva alla selezione" è una feature di qualità a costo implementativo minimo. Il click sinistro è l'interazione principale — immediata, un gesto solo.

---

### [M4] Multi-vault: GitService stateless

**Contesto**: con più vault, `GitService` deve operare su directory diverse. L'implementazione attuale legge `vault.path` una volta nel costruttore e lo usa per tutte le operazioni.

**Decisione**: `GitService` diventa stateless — il path del vault viene passato come parametro a ogni metodo invece di essere memorizzato come campo.

**Motivazione**: più testabile (nessuna dipendenza dal costruttore sulla config), più sicuro in contesti concorrenti (nessuno stato mutabile condiviso), allineato al modello di servizio stateless dei microservizi.

---

### [M4] Separazione vaults.json e current-vault.json

**Contesto**: la configurazione del vault (path, remote, token) e lo stato runtime (vault corrente, timestamp ultimo aggiornamento) hanno cicli di vita diversi e requisiti di sicurezza diversi.

**Decisione**: `vaults.json` contiene la configurazione statica. `current-vault.json` contiene lo stato runtime mutabile. Entrambi esclusi da `.gitignore`. `vaults.json.template` committato come riferimento.

**Motivazione**: configurazione e stato cambiano a velocità diverse. Tenerli insieme esporrebbe le credenziali a scritture frequenti e rischio di corruzione. La separazione è un'abitudine da costruire prima di arrivare ai microservizi, dove config e stato sono sempre tenuti separati.

---

### [M4] AWT invece di Swing per la tray icon

**Contesto**: `SystemTray` e `TrayIcon` sono classi AWT — non hanno equivalente Swing. Il popup del click destro può essere implementato con `PopupMenu` AWT o `JPopupMenu` Swing.

**Decisione**: AWT su tutta la linea. `PopupMenu` per il menu del click destro.

**Motivazione**: look and feel nativo del sistema operativo, zero complessità del bridge AWT→Swing, opzione più stabile su Windows 10 e 11.

**Compatibilità target**: Windows 10 e Windows 11. Windows 8 come bonus opzionale.

---

### [M4] hasUncommittedChanges() come guard per l'autosave invece di hasChanges()

**Contesto**: `hasChanges()` usa `git diff --quiet` — invisibile ai file non tracciati. Un file nuovo appare in `git status` ma non viene committato dall'autosave. Comportamento controintuitivo per qualsiasi sviluppatore familiare con Git.

**Decisione**: sostituire il guard dell'autosave in `SyncOrchestrator` con `hasUncommittedChanges()` che usa `git status --porcelain` — rileva modifiche staged, modifiche non staged e file non tracciati.

**Motivazione**: allineamento con l'aspettativa naturale dello sviluppatore — se `git status` lo vede, l'autosave lo committa. `hasChanges()` viene mantenuto in `GitService` per possibili usi futuri.

---

### [M4] .obsidian/ parzialmente tracciato nei vault

**Contesto**: `.obsidian/workspace.json` salva lo stato dell'interfaccia — pannelli aperti, file attivo, posizione del cursore. Sincronizzarlo tramite `PUSH_LOGOFF` / `PULL_LOGON` significa aprire Obsidian su una seconda macchina esattamente da dove si era rimasti sulla prima.

**Decisione**: tracciare il contenuto di `.obsidian/` ad eccezione dei binari dei plugin. Escludere `plugins/*/main.js` e `plugins/*/styles.css` tramite `.gitignore`.

**Motivazione**: la continuità della sessione è una feature reale consegnata gratuitamente dal meccanismo di sync esistente. I binari dei plugin sono grandi e reinstallabili da Obsidian — nessun valore nel tracciarli.

**Trade-off accettato**: i conflitti su `workspace.json` vengono risolti dalla strategia `theirs` — l'ultima sessione vince sempre.

---

## [M4] push() eseguito sempre al PUSH_LOGOFF, indipendentemente dal commit

**Contesto**: l'autosave committa localmente ad ogni ciclo senza pushare sul remoto.
Al PUSH_LOGOFF ci possono essere N commit locali accumulati da pushare anche quando la
working tree è pulita. Il codice originale saltava il push se `commitLocal()` restituiva
exit code non-zero (niente da committare).

**Decisione**: `push()` viene chiamato sempre nel case PUSH_LOGOFF e PUSH_MANUAL,
indipendentemente dall'exit code di `commitLocal()`.

**Motivazione**: "niente da committare" e "niente da pushare" sono condizioni distinte.
L'autosave accumula commit locali durante la sessione — il logoff è il momento in cui
vengono trasmessi al remoto. Saltare il push in assenza di nuove modifiche locali
vanificava l'intera pipeline autosave → push al logoff.


---

## Obiettivi dello sprint — cosa entra nel done

|#|Obiettivo|Criterio di accettazione|
|---|---|---|
|1|`NetworkException` / `GitException` in `GitService`|whitelist pattern di rete, catch separati in `SyncOrchestrator`|
|2|Modalità client/server nel JAR|`tray` avvia server, `logon`/`logoff`/`autosave` inviano evento via socket|
|3|Socket server su `localhost:4242`|riceve JSON, pubblica evento su `SyncOrchestrator`|
|4|Socket client con retry|exponential backoff 30s→60s→120s, max 3 tentativi|
|5|`GitService` stateless multi-vault|vault passato per parametro a tutti i metodi|
|6|`vaults.json` + `vaults.json.template`|struttura definita, escluso da `.gitignore`|
|7|`current-vault.json`|scritto al primo avvio, aggiornato al cambio vault|
|8|Tray icon — click sinistro|pubblica `PUSH_MANUAL` sul vault corrente|
|9|Tray icon — click destro|popup con elenco vault + boolean "salva alla selezione"|
|10|Task Scheduler — 3 task configurati|logon, logoff, autosave avviano il JAR con argomento corretto|
|11|Checklist e2e superata|tutti gli 8 scenari validati su vault di test|

---

## Rischi identificati

|Rischio|Probabilità|Impatto|Mitigazione|
|---|---|---|---|
|Messaggi stderr Git localizzati in italiano su Windows|Media|Alto|testare sulla macchina reale prima di finalizzare la whitelist|
|Porta 4242 occupata su alcune macchine|Bassa|Medio|porta configurabile via `config.properties`|
|Tray icon AWT su Windows 11|Media|Alto|testare `SystemTray.isSupported()` al bootstrap; Windows 10 è il target primario|
|Scrittura concorrente su `current-vault.json`|Media|Medio|sincronizzare le scritture con `synchronized`|
|Task Scheduler logoff non attende completamento JAR|Media|Alto|configurare il task con "attendi completamento"|
|Remote URL inesistente non ripristinato dopo crash nel test e2e|Bassa|Medio|script di teardown esplicito nella checklist|

---

## Know-how da acquisire prima di scrivere

| Area                   | Concetto                                                    | Perché serve                                 |
| ---------------------- | ----------------------------------------------------------- | -------------------------------------------- |
| Java Networking        | `ServerSocket`, `Socket`, `BufferedReader`, `PrintWriter`   | socket server e client                       |
| Java AWT               | `SystemTray`, `TrayIcon`, `PopupMenu`, `MenuItem`           | tray icon con click sinistro/destro          |
| Java AWT               | Event Dispatch Thread, `SwingUtilities.invokeLater()`       | thread safety tra tray e orchestratore       |
| JSON in Java           | `org.json` o `Jackson` — parsing e serializzazione          | protocollo socket e lettura `vaults.json`    |
| Windows Task Scheduler | trigger logon/logoff, opzioni di esecuzione, account utente | configurazione dei 3 task                    |
| Git internals          | parsing stderr, pattern di errore di rete                   | implementazione whitelist `NetworkException` |


---

# EXTRA DTR


---

## [M4] PULL_MANUAL — manual pull from tray icon 
**Contesto**: un utente con due macchine sempre accese non ottiene mai un PULL_LOGON automatico. Le modifiche pushate da un dispositivo non raggiungono l'altro finché non viene eseguito un pull esplicito. 
**Decisione**: aggiungere EventType.PULL_MANUAL con priorità 1. Accessibile tramite icona refresh per vault nel popup del click destro sulla tray icon. 
**Motivazione**: PULL_LOGON risolve il caso normale. PULL_MANUAL risolve il caso di sessioni persistenti — macchine mai spente, server sempre attivi, utenti che lavorano su più dispositivi senza logoff. 
**Impatto**: minimo lato backend (30 min). Lato tray: integrato naturalmente nel popup multi-vault già pianificato.

---

## [M4] Caller logging rimandato a framework di logging maturo

**Contesto**: durante il debugging di `mvn test`, è emersa la necessità di identificare
programmaticamente il chiamante in ogni riga di log — per distinguere da quale test o
componente proviene un determinato messaggio senza analisi manuale dello stack.

**Decisione**: non implementare il caller logging in `LogService`. Rimandato.

**Motivazione**: `Thread.currentThread().getStackTrace()` è nativo Java e tecnicamente
fattibile, ma introduce rumore in produzione — ogni riga porta il nome del chiamante
anche quando non è utile. La soluzione standard è un framework di logging maturo
(slf4j + logback) con MDC (Mapped Diagnostic Context), che gestisce questa funzionalità
in modo configurabile per ambiente. Aggiungere una soluzione custom ora significherebbe
riscriverla quando si adotterà logback nei microservizi.

**Principio**: KISS — Better Done Than Perfect. Il valore non giustifica l'effort
in questa fase del progetto.

**Prossimo step**: quando ObsidianSync verrà integrato nell'architettura a microservizi
di ToDoList 2.0, `LogService` verrà sostituito da slf4j + logback. Il caller logging
sarà disponibile gratuitamente tramite MDC senza nessuna riga di codice aggiuntiva.

---

## [M4] Architettura multi-vault con VaultContext e coda globale prioritizzata

**Contesto**: ObsidianSync nasce come tool mono-vault. L'introduzione del layer socket
(SocketServer/SocketClient) ha fatto emergere la necessità di gestire vault multipli
con isolamento delle code ed ordinamento cross-vault degli eventi.

**Decisione**: ogni vault registrato ottiene una `SyncEventQueue` dedicata e un thread
aggregatore, incapsulati in `VaultContext`. Una `PriorityBlockingQueue<SyncEvent>` globale
raccoglie gli eventi da tutte le code per-vault e li consegna al worker in ordine di
priorità cross-vault.

**Motivazione**: isolamento per vault — un vault lento non blocca gli altri. Priorità
globale garantita per costruzione — un `PULL_LOGON` su vault A precede sempre un
`AUTOSAVE` su vault B indipendentemente dall'ordine di arrivo.

**Alternativa scartata**: `vaultId` come campo di routing in una coda unica senza
isolamento — più semplice ma senza garanzie di isolamento tra vault.

---

## [M4] Composizione su ereditarietà per VaultContext

**Contesto**: `VaultContext` deve accedere ai campi di `Vault` (id, name, path) e
aggiungere stato runtime (coda, thread aggregatore, future). L'ereditarietà avrebbe
mescolato un value object immutabile con stato mutabile di esecuzione.

**Decisione**: `VaultContext` contiene `Vault` come campo `final` — composizione.

**Motivazione**: `Vault` è un value object deserializzato da JSON con ciclo di vita
distinto da quello del runtime. Tenerli separati è un'abitudine da costruire prima
di arrivare ai microservizi, dove configurazione e stato sono sempre in classi distinte.

---

## [M4] Layer DTO per il parsing JSON — SocketMessageDto

**Contesto**: la deserializzazione Jackson richiede costruttore no-arg o `@JsonCreator`.
Applicare annotazioni Jackson a `SyncEvent` avrebbe introdotto una dipendenza dal layer
di trasporto nel modello di dominio — campanello d'allarme architetturale.

**Decisione**: introdurre `package dto` con `SocketMessageDto` annotato con
`@JsonCreator` e `@JsonProperty`. `JsonMapper` converte il DTO in `SyncEvent`
senza esporre Jackson al dominio.

**Motivazione**: separazione netta tra contratto di trasporto e modello di dominio.
Il pattern è identico a quello usato in Spring Boot con i DTO di request/response.

---

## [M4] JsonMapper come utility layer di serializzazione

**Contesto**: la logica di serializzazione/deserializzazione Jackson era dispersa tra
`SocketServer`, `SocketClient` e classi di dominio.

**Decisione**: centralizzare in `JsonMapper` con metodi statici — una singola istanza
`ObjectMapper` condivisa, thread-safe dopo la configurazione iniziale.

**Motivazione**: `ObjectMapper` è costoso da istanziare — creare un'istanza per chiamata
è un anti-pattern documentato. La centralizzazione elimina la duplicazione e garantisce
un solo punto di configurazione Jackson.

---

## [M4] SocketServer — separazione receiver e router

**Contesto**: il server deve accettare connessioni in entrata (bloccante su `accept()`)
e instradare gli eventi sulla coda per-vault (bloccante su `take()`). Eseguire entrambe
le operazioni sullo stesso thread avrebbe serializzato ricezione e routing.

**Decisione**: due thread dedicati — `receiver` (loop su `accept()`) e `router`
(loop su `mainQueue.take()`). Comunicano tramite `mainQueue`.

**Motivazione**: disaccoppiamento tra IO di rete e logica di instradamento. Il receiver
non si blocca sul routing; il router non si blocca sull'accettazione di nuove connessioni.
`mainQueue.take()` invece di `poll()` + sleep elimina il busy-waiting.

---

## [M4] AUTOSAVE come evento broadcast con vaultId null

**Contesto**: `AutosaveScheduler` pubblica eventi periodici senza conoscere la lista
dei vault registrati. Passare il vaultId al costruttore dello scheduler richiedeva
un aggiornamento ad ogni aggiunta o rimozione di vault a runtime.

**Decisione**: `AUTOSAVE` viene pubblicato con `vaultId = null` come sentinella di
broadcast. `SocketServer.doWork()` espande l'evento in uno per vault tramite
`SyncEvent.forVault(String)` prima di pubblicare sulle code per-vault.

**Motivazione**: `AutosaveScheduler` rimane ignaro dei vault — zero accoppiamento
con la lista vault, zero aggiornamenti runtime. Il broadcast è una responsabilità
del layer di routing, non del publisher.

**Trade-off accettato**: `doWork` deve distinguere eventi broadcast da eventi
targetizzati tramite null check su `vaultId`. Documentato nel javadoc di `SyncEvent`
e `SocketServer`.

---

## [M4] SyncEvent.forVault() — copia targetizzata di un evento broadcast

**Contesto**: il broadcast richiede di creare N eventi targetizzati a partire da
uno broadcast, preservando timestamp e retryDelay originali per garantire l'ordinamento
cross-vault consistente.

**Decisione**: metodo `forVault(String vaultId)` su `SyncEvent` che restituisce
una copia con il vaultId assegnato. Lancia `UnsupportedOperationException` se
l'evento ha già un vaultId — guard difensivo contro uso improprio.

**Motivazione**: la copia preserva il timestamp originale — tutti gli eventi generati
da un broadcast condividono la stessa priorità temporale, garantendo fairness
cross-vault nell'ordinamento della coda.

---

## [M4] SocketServer — readLine() invece di extractJson() per il protocollo request-response

**Contesto**: il receiver usava `JsonMapper.extractJson()` basato sul parser streaming
di Jackson. Jackson tiene lo stream aperto finché non vede EOF — in un protocollo
request-response su socket il client non chiude la connessione prima di aver letto
la risposta, generando un deadlock.

**Decisione**: sostituire `extractJson()` con `BufferedReader.readLine()` nel receiver.
Il client invia una singola riga JSON terminata da `\n` via `PrintWriter.println()` —
`readLine()` si ferma al newline senza aspettare la chiusura del socket.

**Motivazione**: il protocollo è già line-based — client e server si scambiano una
riga per direzione. `readLine()` è la primitiva corretta per questo pattern.
`extractJson()` rimane disponibile per contesti dove si legge da stream senza
protocollo line-based.

---

## [M4] SocketServer — ACK prima del parsing vs dopo

**Contesto**: la prima implementazione mandava ACK dopo aver messo l'evento sulla
mainQueue. Il client leggeva null perché il server chiudeva il socket nel finally
prima che l'ACK fosse inviato.

**Decisione**: il server risponde ACK/NACK/ERROR prima di chiudere il socket, con
il `finally` che chiude il socket solo dopo che la risposta è stata scritta.

**Motivazione**: il protocollo garantisce che ogni connessione riceva esattamente
una risposta prima della chiusura. Il `PrintWriter` con `autoFlush=true` assicura
che la risposta sia sul filo prima del `socket.close()`.

---

## [M4] SocketServer — tre livelli di risposta ACK/NACK/ERROR

**Contesto**: il server deve distinguere tra evento sconosciuto (errore semantico)
e JSON malformato (errore sintattico). Trattarli allo stesso modo impedirebbe al
client di capire la natura del problema.

**Decisione**:
- `ACK` — evento parsato e accodato con successo
- `NACK` — `IllegalArgumentException` su `EventType.valueOf()` — tipo evento non riconosciuto
- `ERROR` — `JsonProcessingException` — JSON malformato

**Motivazione**: separazione delle responsabilità a livello di protocollo.
Il client può reagire diversamente ai tre casi — retry su ERROR, nessun retry su NACK.

---

## [M4] SyncOrchestrator — uno per vault, avviato via ScheduledFuture

**Contesto**: l'architettura mono-orchestratore non era scalabile al multi-vault.
La proposta di un aggregatore per vault che faceva da ponte verso una coda globale
creava cicli logici (coda → aggregatore → coda).

**Decisione**: un `SyncOrchestrator` per vault, avviato con `scheduler.schedule(orchestrator::start, 0, MILLISECONDS)`. Il `ScheduledFuture` restituito viene conservato in `VaultContext` per `cancel(true)` allo shutdown.

**Motivazione**: ogni vault ha la sua coda isolata e il suo worker seriale. La
priorità cross-vault è garantita dalla `PriorityBlockingQueue` globale in
`SocketServer`. Il pattern `ScheduledFuture` centralizza la gestione del ciclo
di vita dei thread senza introdurre aggregatori intermedi.

**Alternativa scartata**: aggregatore per vault che consuma dalla coda per-vault
e mette sulla mainQueue — ciclo logico, complessità aggiuntiva senza benefici.