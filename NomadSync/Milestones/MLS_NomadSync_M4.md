# ObsidianSync — Report Milestone 4

## Obiettivi

Introdurre il layer di comunicazione TCP tra Task Scheduler e processo tray, abilitare l'architettura multi-vault con isolamento per-vault, completare la suite di test a copertura completa.

---

## Deliverable

|Componente|Stato|
|---|---|
|`SocketClient`|✅|
|`SocketServer`|✅|
|`SocketMessage` / `SocketMessageDto`|✅|
|`SocketResponse`|✅|
|`Vault`|✅|
|`VaultContext`|✅|
|`VaultService`|✅|
|`JsonMapper`|✅|
|`VaultDto` / `VaultRootDto`|✅|
|`AutosaveScheduler` (refactoring broadcast)|✅|
|`SyncEvent` (vaultId + forVault)|✅|
|`SocketClientTest`|✅|
|`SocketServerTest`|✅ (6 test)|
|`VaultServiceTest`|✅ (14 test)|
|Test suite complessiva|✅ **61 test, 0 failure, ~8s**|

---

## Architettura implementata

```
SocketClient (Task Scheduler)
    ↓ TCP — JSON line
SocketServer.doReceive()
    ↓ mainQueue.offer()
SocketServer.doRedirect()
    ↓ vaultQueue.publish() [broadcast se vaultId=null]
VaultContext.queue (per vault)
    ↓ queue.consume()
SyncOrchestrator.worker (per vault)
    ↓
GitService (stateless, vaultPath param)
```

### Decisioni chiave

**Multi-vault**: un `SyncOrchestrator` per vault avviato via `ScheduledFuture`. Ogni vault ha la sua `SyncEventQueue` isolata. La `PriorityBlockingQueue` globale garantisce priorità cross-vault.

**Protocollo socket**: line-based — `PrintWriter.println()` lato client, `BufferedReader.readLine()` lato server. Tre livelli di risposta: ACK / NACK / ERROR.

**Broadcast AUTOSAVE**: `AutosaveScheduler` pubblica `SyncEvent(AUTOSAVE, null)`. `doRedirect()` espande a tutti i vault registrati via `SyncEvent.forVault()`.

**DTO layer**: `SocketMessageDto`, `VaultDto`, `VaultRootDto` separano Jackson dal dominio. `Vault` e `SyncEvent` non dipendono da annotazioni Jackson.

**`VaultService`**: `HashMap<String, Vault>` per O(1) lookup. Ogni mutazione persiste su `vaults.json` via `JsonMapper`.

---

## Problemi affrontati

| Problema                           | Causa                                                                | Soluzione                                                                                    |
| ---------------------------------- | -------------------------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| Deadlock receiver                  | `extractJson()` aspetta EOF                                          | `readLine()` — line-based protocol                                                           |
| Socket chiuso prima dell'ACK       | `try-with-resources` chiudeva il socket                              | `finally` esplicito, ACK prima della chiusura                                                |
| `IllegalThreadStateException`      | `start()` chiamato due volte sullo stesso thread                     | `scheduler.schedule()` una sola volta per vault                                              |
| Test bloccato su `queue.consume()` | Coda sbagliata — quella del test, non del VaultContext               | Rimossa asserzione ridondante, `await` su mock                                               |
| `gitService.pull()` mai chiamato   | `register()` chiamato prima di `start()` — orchestratore non avviato | `scheduler.schedule()` avvia l'orchestratore in `register()`, indipendentemente da `start()` |
