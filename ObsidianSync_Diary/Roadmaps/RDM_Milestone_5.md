## Roadmap M5 — ordine implementativo

Tre layer in sequenza. Niente UI finché il backend non regge.

---

### Layer 1 — Refactoring backend (prerequisiti bloccanti)

**1. `EventType`** — rimozione `PULL_MANUAL` e `PUSH_MANUAL`, aggiunta `SYNCHRONIZE` con priorità 2. Impatta tutta la suite — da fare per prima.

**2. `GitService`** — nuovo metodo `synchronize(vaultPath)` che implementa l'algoritmo SYNCHRONIZE del GRM: commit locale se dirty → pull → se conflitto: abort, backup FIFO, pull -X ours, snapshot remote-conflicts, push. Questo è il metodo più complesso della milestone.

**3. `SyncOrchestrator`** — aggiornamento switch per gestire `SYNCHRONIZE`, rimozione case `PULL_MANUAL`/`PUSH_MANUAL`.

**4. `VaultService`** — aggiunta vincolo unicità `vault.name` con `VaultException` su duplicato. Validazione all'avvio.

**5. `LogService`** — refactoring multi-vault: il log deve includere il vault corrente nel contesto. Ogni `VaultContext` ha il suo `LogService` o il `LogService` condiviso riceve il `vaultId` come parametro.

---

### Layer 2 — TrayManager

**6. `TrayManager`** — `SystemTray` + `TrayIcon` AWT. Costruito in tre sottopassi:

- **6a** `TrayIcon` base — icona statica, tooltip, click sinistro pubblica `SYNCHRONIZE` sul vault corrente
- **6b** `ContextMenu` — struttura completa del menu tasto destro
- **6c** `VaultSwitcher` — sottomenu vault con checkbox "Save on selection"

**7. `ToastNotification`** — tre scenari (success, conflitto, fallimento rete). `displayMessage()` di AWT per i toast semplici; finestra custom `JDialog` per i toast azionabili con bottone "Apri versioni remote".

---

### Layer 3 — Main + integrazione

**8. `Main`** — modalità tray: carica `vaults.json`, costruisce `SocketServer`, registra i vault, avvia `TrayManager`, blocca sul `SystemTray`.

**9. Task Scheduler Windows** — 3 task: Pull@logon (`PULL_LOGON` via `SocketClient`), Push@logoff (`PUSH_LOGOFF`), Autosave@15min.

**10. Test e2e manuale** — vault reale, due sessioni logon/logoff.

**11. Release 1.0.0** — `git flow release finish 1.0.0`.

---

### Scope rimandato a M6

`MainWindow` (struttura tab completa), `PrismUI` design system Maven, i18n 10 lingue — fuori scope M5, troppo volume.

---

### Ordine di attacco per la sessione di stamattina

Inizia da `EventType` — è la modifica più piccola con l'impatto più grande sulla suite. Quando `mvn test` è verde con `SYNCHRONIZE` al posto di `PULL_MANUAL`/`PUSH_MANUAL`, tutto il resto del Layer 1 ha una base stabile.