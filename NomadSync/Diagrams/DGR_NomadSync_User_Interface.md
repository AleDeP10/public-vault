# User Interface

## Componenti UI — elenco definitivo

**`TrayIcon`** — l'icona nel System Tray (vicino all'orologio). Cambia stato visivo: idle, syncing, error. Click sinistro → `PUSH_MANUAL` sul vault corrente con Toast di conferma.

**`ContextMenu`** — menu tasto destro sulla `TrayIcon`. Accesso rapido alle operazioni più frequenti, sempre visibile, minimale.

**`MainWindow`** — finestra principale dell'applicazione. Accesso completo a tutte le funzionalità, impostazioni e log. Si apre da ContextMenu o da doppio click sull'icona.

**`ToastNotification`** — notifica a scomparsa di Windows. Feedback su push/pull completati, errori, conflitti risolti. Versione azionabile con bottone "Apri cartella" per i conflitti.

**`VaultSwitcherPanel`** — pannello di selezione vault nel ContextMenu. Lista vault disponibili + checkbox "Save on selection".

---

Ecco il diagramma delle interazioni:
![[IMG_NomadSync_UI_Architecture.png]]
Il `VaultSwitcherPanel` è figlio del `ContextMenu` ma aggiorna lo stato in `current-vault.json` e pubblica direttamente sull'orchestratore se "Save on selection" è attivo — è il componente con più responsabilità nel menu rapido.


---

## Tray Icon

La `TrayIcon` è il cuore visibile di ObsidianSync — il punto di contatto principale tra l'utente e il sistema.

**Posizione**: System Tray di Windows, vicino all'orologio in basso a destra. Non nella taskbar — è un servizio di background, non un'app attiva.

**Stato visivo**: l'icona cambia apparenza per comunicare lo stato del sistema senza che l'utente debba aprire nulla:

| Stato    | Icona             | Quando                                            |
| -------- | ----------------- | ------------------------------------------------- |
| Idle     | verde / statica   | nessuna operazione in corso                       |
| Syncing  | animata / rotante | pull o push in esecuzione                         |
| Error    | rossa             | ultimo sync fallito dopo 3 retry                  |
| Conflict | arancione         | file in `remote-conflicts` in attesa di revisione |

**Click sinistro**: pubblica `PUSH_MANUAL` sul vault corrente direttamente sull'orchestratore. Toast di conferma al completamento.

**Click destro**: apre il `ContextMenu`.

**Tooltip**: al passaggio del mouse mostra "Last sync: X minutes ago" — informazione sempre disponibile senza aprire nulla.

---

Tecnicamente è implementata con `SystemTray` e `TrayIcon` di Java AWT — le uniche API Java che permettono l'integrazione con la notification area di Windows. Lo stato visivo si aggiorna tramite `trayIcon.setImage()` con immagini pre-caricate per ogni stato.




---

## ContextMenu

Il `ContextMenu` è il **pannello di controllo rapido** di ObsidianSync — appare con un click destro sulla `TrayIcon` e raccoglie tutte le operazioni che un utente esegue durante una sessione normale, senza dover aprire la `MainWindow`.

È progettato attorno a un principio: **zero decisioni cognitive**. Ogni voce ha un nome che descrive esattamente cosa fa, raggruppata con le voci affini. L'utente non deve sapere cosa sia un pull o un push — vede "Sync current vault" e capisce.

La voce in cima mostra sempre il vault attivo con una freccia — passandoci sopra si apre il sottomenu per cambiarlo. Sotto, le tre azioni di sync coprono tutti i casi d'uso senza sovrapposizioni. Il resto è accesso rapido al log, alla cartella vault, e alla `MainWindow` per chi vuole andare più in profondità.

### Granularità totale
┌─────────────────────────────┐
│  ● Personal             ▶   │  → sottomenu vault switcher
├─────────────────────────────┤
│  Sync current vault         │  → PULL + PUSH su current vault
│  Sync all vaults            │  → PULL + PUSH broadcast
│  Pull current vault         │  → PULL_MANUAL su current vault
├─────────────────────────────┤
│  Last sync: 3 min ago       │  (label, non cliccabile)
│  View log                   │  → tab Log in MainWindow
├─────────────────────────────┤
│  Open Dashboard             │  → MainWindow
│  Open vault folder          │  → Explorer sulla cartella vault
├─────────────────────────────┤
│  Exit                       │
└─────────────────────────────┘

Sottomenu vault (hover su prima voce):
┌─────────────────┐
│ ✓ Personal      │
│   Work          │
│   Research      │
│ ─────────────── │
│ ☐ Save on sel.  │
└─────────────────┘


---


## ToastNotification — Bottone remote-conflicts

### Il caso d'uso

Il toast appare dopo un `PULL_MANUAL` con conflitti risolti via `-X ours`. L'utente sa che i suoi file locali sono stati preservati, ma c'è lavoro remoto disponibile in `remote-conflicts/` che potrebbe voler integrare manualmente.

Il bottone deve portarlo **direttamente** alla cartella giusta — senza passare per Explorer, senza cercare il path a mano.

---

### Implementazione

Java AWT espone `Desktop.getDesktop().open(File)` — apre la cartella nel file manager nativo del sistema operativo. Su Windows apre Explorer, su macOS apre Finder. Zero dipendenze aggiuntive, comportamento nativo garantito.

java

```java
Desktop.getDesktop().open(
    new File("ObsidianSync/remote-conflicts/Personal_2026-05-15_20-40/")
);
```

Il toast mostra il bottone solo quando esiste effettivamente una cartella `remote-conflicts` con contenuto — se il pull è andato liscio senza conflitti, il bottone non compare.

---

### Struttura del toast nei tre scenari

**Sync completato senza conflitti** — toast minimo, scompare da solo:

```
✓ Personal sincronizzato
```

**Sync completato con conflitti risolti** — toast persistente, richiede dismiss:

```
⚠ Sync completato con conflitti

File aggiornati dal remoto:
  • Gabriela_experiment.md
  • note-lavoro.md

La tua versione locale è stata mantenuta.
Le versioni remote sono disponibili per il confronto.

[ Apri versioni remote ]     [ OK ]
```

**Sync fallito dopo 3 retry** — toast persistente, solo per priorità 1:

```
✗ Pull non riuscito

Nessuna connessione disponibile dopo 3 tentativi.
Il vault potrebbe non essere aggiornato.

[ OK ]
```

---

### Un dettaglio UX importante

Il bottone "Apri versioni remote" apre la cartella `remote-conflicts/Personal_{timestamp}/` — quella dell'ultimo conflitto, non la root di `remote-conflicts/`. L'utente si trova direttamente davanti ai file da confrontare, non deve navigare ulteriormente.

Il timestamp nel nome della cartella è in formato leggibile (`2026-05-15_20-40`) — se l'utente ha più snapshot, li vede ordinati cronologicamente in Explorer senza bisogno di metadati aggiuntivi.

---

### Toast aggiornato con link secondario


⚠ Sync completato con conflitti

File aggiornati dal remoto:
  • Gabriela_experiment.md
  • note-lavoro.md

La tua versione locale è stata mantenuta.

[ Apri versioni remote ]

Hai perso delle modifiche? → Apri backup locale


---

### Struttura MainWindow — proposta completa

Prima il testo, poi il wireframe.

**Apertura contestuale** confermata:

- Da "View log" → MainWindow apre direttamente tab Log, vault pre-selezionato
- Da "Conflicts" → tab Conflict, vault pre-selezionato
- Da TrayIcon / eseguibile → Home dashboard

**Vault switcher** in cima — `ComboBox` autocomplete, sempre visibile in tutti i tab. Cambiare vault aggiorna tutto il contenuto sottostante.

**Tab**:

- **Home** — dashboard con card per vault, alerts, attività recente
- **Properties** — form di editing `vaults.json` per il vault selezionato
- **Log** — visualizzatore eventi stile IntelliJ/Grafana
- **Conflicts** — file in `remote-conflicts/` con bottone apri + stato risolto/aperto
- **Backup** — lista snapshot FIFO con bottone apri cartella
- **Settings** — properties globali: porta socket, lingua, tema
![[IMG_NomadSync_MainWindow_Settings.png]]

---
### Conferma di risoluzione conflict

Hai copiato le modifiche remote che volevi preservare?

Una volta confermato, la versione remota di
note-lavoro.md verrà eliminata.

[ Annulla ]    [ Sì, ho finito ]