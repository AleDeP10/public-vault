# GRM — Milestone 4: Integrazione Windows

---

## Contesto

Questa milestone completa l'integrazione con Windows e introduce le feature visibili all'utente: tray icon, multi-vault, push manuale. È l'ultimo sprint prima della validazione end-to-end su vault reale.

---

## Sessione di Grooming

### NetworkException vs GitException

Git restituisce exit code generici — non distingue errori di rete da errori locali. La distinzione va ricavata analizzando lo stderr. Tre approcci valutati:

- **Pattern matching generico** — rischio falsi positivi sul retry
- **Exit code + stderr combinati** — exit code 128 non è esclusivo degli errori di rete
- **Whitelist dei pattern di rete** ✅ — approccio conservativo, solo i pattern noti lanciano `NetworkException`, tutto il resto è `GitException`

Pattern riconosciuti come errori di rete: `timeout`, `Could not resolve host`, `Connection refused`, `Failed to connect`, `Network is unreachable`.

**Rischio identificato**: su installazioni Windows in italiano, i messaggi stderr di Git potrebbero essere localizzati. Da verificare sulla macchina reale prima di finalizzare la whitelist.

---

### Processo persistente e Task Scheduler

Task Scheduler avvia una nuova JVM per ogni task — non c'è un orchestratore persistente condiviso tra i processi. Due modelli valutati:

- **Processo effimero per task** — semplice, ma la coda a priorità perde senso
- **Processo persistente con IPC** ✅ — un processo principale ospita l'orchestratore, i task sono client che gli inviano eventi

La tray icon è il candidato naturale come processo host: è già persistente per natura, già prevista nell'architettura, e consolida orchestratore e UI nello stesso processo.

**Modalità di avvio del JAR**:

- `tray` → avvia tray + socket server + orchestratore
- `logon` / `logoff` / `autosave` → client socket, inviano evento e terminano

---

### IPC: Local Socket

Tre opzioni valutate per la comunicazione tra task e processo principale:

- **File di lock** — polling, non bidirezionale, non scala
- **Named pipe Windows** — API Java macchinose, non portabile su macOS
- **Local socket TCP** ✅ — bidirezionale, portabile, base concettuale per WebSocket e microservizi

Porta scelta: **4242**, configurabile via `config.properties`.

La scelta è motivata anche dall'obiettivo formativo: il modello concettuale del socket TCP è identico a quello delle WebSocket. Chi capisce uno capisce l'altro.

---

### Protocollo messaggi: JSON

Stringa semplice scartata — non estensibile, nessun supporto a metadati futuri. JSON adottato con campi `event`, `vaultId`, `timestamp`. Il campo `vaultId` è già predisposto per il multi-vault.

---

### Retry client socket

Il client socket adotta la stessa politica di retry dell'orchestratore — exponential backoff 30s → 60s → 120s, massimo 3 tentativi. Nessun delay esplicito tra task in Task Scheduler: i 30 secondi del primo tentativo coprono qualsiasi finestra di startup su un sistema contemporaneo.

---

### PUSH_MANUAL e tray icon

`PUSH_MANUAL` è la feature più visibile all'utente. La tray icon si comporta così:

- **Click sinistro** → `PUSH_MANUAL` sul vault corrente, pubblicato direttamente sull'orchestratore interno (nessun IPC — stessa JVM)
- **Click destro** → popup con elenco vault disponibili + boolean "Save on selection"
    - Selezione vault → aggiorna `current-vault.json`
    - Se "Save on selection" attivo → pubblica automaticamente `PUSH_MANUAL` sul vault appena selezionato

"Save on selection" valutato come feature di qualità a costo implementativo minimo — non sovraingegneria.

---

### Multi-vault

Struttura `vaults.json` come registro centralizzato. Stessa politica di sicurezza di `config.properties`: file escluso da `.gitignore`, template committato.

`GitService` diventa stateless — il vault viene passato come parametro ad ogni metodo. Più testabile, più sicuro in contesti concorrenti, allineato ai microservizi.

---

### Separazione configurazione e stato

`vaults.json` contiene la configurazione statica (path, remote, token). `current-vault.json` contiene lo stato mutabile (vault corrente, timestamp aggiornamento). Tenere i due insieme esporrebbe le credenziali a scritture frequenti e rischio di corruzione.

Principio: configurazione e stato hanno cicli di vita diversi. Separarli è un'abitudine da costruire prima di arrivare ai microservizi.

---

### AWT vs Swing per la tray

`SystemTray` e `TrayIcon` sono classi AWT — non hanno equivalente Swing. Il popup del click destro può essere implementato con `PopupMenu` AWT o `JPopupMenu` Swing.

AWT puro confermato: look nativo, integrazione con il sistema operativo, zero complessità aggiuntiva per il bridge AWT→Swing. Su Windows 10 e 11 è la scelta più stabile.

**Compatibilità target**: Windows 10 e Windows 11. Windows 8 come bonus opzionale.

---

### Test end-to-end

Checklist manuale, eseguita una volta prima del rilascio definitivo. Repo GitHub dedicata ai test, vault locale dedicato, token con scope limitato. Quando tutti gli 8 scenari passano, il progetto è done. Automazione sproporzionata rispetto agli obiettivi.

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
|9|Tray icon — click destro|popup con elenco vault + boolean "save on selection"|
|10|Task Scheduler — 3 task configurati|logon, logoff, autosave avviano il JAR con argomento corretto|
|11|Checklist e2e superata|tutti gli 8 scenari validati su vault di test|

---

## Rischi identificati — cosa potrebbe bloccare

|Rischio|Probabilità|Impatto|Mitigazione|
|---|---|---|---|
|Messaggi stderr Git localizzati in italiano|media|alto|testare sulla macchina reale prima di finalizzare la whitelist|
|Porta 4242 occupata su alcune macchine|bassa|medio|porta configurabile via `config.properties`|
|Tray icon AWT su Windows 11|media|alto|testare `SystemTray.isSupported()` al bootstrap; Windows 10 è il target primario|
|Scrittura concorrente su `current-vault.json`|media|medio|sincronizzare le scritture con `synchronized`|
|Task Scheduler logoff non attende completamento JAR|media|alto|configurare il task con "wait for task to complete"|
|Remote URL inesistente non ripristinato dopo crash nel test e2e|bassa|medio|script di teardown esplicito nella checklist|

---

## Know-how da acquisire — cosa studiare prima di scrivere

| Area                   | Concetto                                                    | Perché serve                                 |
| ---------------------- | ----------------------------------------------------------- | -------------------------------------------- |
| Java Networking        | `ServerSocket`, `Socket`, `BufferedReader`, `PrintWriter`   | socket server e client                       |
| Java AWT               | `SystemTray`, `TrayIcon`, `PopupMenu`, `MenuItem`           | tray icon con click sinistro/destro          |
| Java AWT               | Event Dispatch Thread, `SwingUtilities.invokeLater()`       | thread safety tra tray e orchestratore       |
| JSON in Java           | `org.json` o `Jackson` — parsing e serializzazione          | protocollo socket e lettura `vaults.json`    |
| Windows Task Scheduler | trigger logon/logoff, opzioni di esecuzione, account utente | configurazione dei 3 task                    |
| Git internals          | stderr parsing, pattern di errore di rete                   | implementazione whitelist `NetworkException` |