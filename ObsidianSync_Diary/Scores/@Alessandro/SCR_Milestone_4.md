# Pagelle ottenute durante Milestone 4

---

## Pagella — aggiornamento test

**Voto: 9.5/10**

| Dimensione         | Voto  | Note                                                                                                                                                                                          |
| ------------------ | ----- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Rapidità delivery  | 10/10 | Suite aggiornata autonomamente, tutti e 4 i nuovi casi identificati e implementati                                                                                                            |
| Correttezza logica | 9/10  | Copertura completa: dirty/clean × commit/no commit × PUSH_LOGOFF/PUSH_MANUAL                                                                                                                  |
| Autonomia          | 10/10 | Ha recepito il feedback sui casi mancanti e li ha aggiunti senza imbeccate                                                                                                                    |
| Manutenibilità     | 9/10  | Un punto migliorabile: `pushManual_withChanges_commitsAndPushes` e `pushLogoff_withChanges_commitsAndPushes` sono identici — candidati a parametrizzazione con `@ParameterizedTest` in futuro |

## Pagella — refactor eccezioni

**Voto: 8.5/10**

| File               | Voto   | Note principali                                                                                             |
| ------------------ | ------ | ----------------------------------------------------------------------------------------------------------- |
| `CommandUtil`      | 8/10   | `NETWORK_PATTERNS` non `final`, pattern matching su `e.getMessage()` potrebbe NPE se il messaggio è null    |
| `GitService`       | 8/10   | Catch `NetworkException` silenzioso su operazioni locali — la domanda nei commenti è giusta, risposta sotto |
| `GitServiceTest`   | 9/10   | Signature corrette, isolamento mantenuto, nessun test broken                                                |
| `SyncOrchestrator` | 8.5/10 | Domanda nei commenti giusta — risposta sotto                                                                |

## Pagella — Introduzione PULL_MANUAL

**Voto: 8.7/10**

| Dimensione             | Voto | Note                                                                                      |
| ---------------------- | ---- | ----------------------------------------------------------------------------------------- |
| Rapidità delivery      | 9/10 | `PULL_MANUAL` implementato in autonomia in meno di 30 minuti come stimato                 |
| Correttezza logica     | 9/10 | Case unificato con `PULL_LOGON` corretto; priorità 1 corretta                             |
| Autonomia              | 9/10 | Ha identificato il caso d'uso reale, proposto la soluzione e implementato senza imbeccate |
| Qualità del codice     | 8/10 | `vaultPath` nei metodi anticipa il multi-vault — scelta lungimirante ma ahead of schedule |
| Visione architetturale | 9/10 | La proposta tray con icona refresh per vault è coerente e ben motivata                    |


---


## SCR — Sprint SocketClient

### Punteggio complessivo: 8.8 / 10

| Dimensione                        | Voto | Note                                                                                                                 |
| --------------------------------- | ---- | -------------------------------------------------------------------------------------------------------------------- |
| Qualità del codice prodotto       | 9/10 | NACK→IOException è una soluzione elegante che riusa il meccanismo esistente senza duplicare logica                   |
| Comprensione dei requisiti        | 9/10 | Ha anticipato PULL_MANUAL e il caso d'uso multi-sessione senza che venisse richiesto                                 |
| Autonomia tecnica                 | 9/10 | ServerSocket(0), Runnable pluggabile, TestProperties — tutti trovati autonomamente                                   |
| Testing                           | 8/10 | Pattern serverBehavior + startServer() pulito; retriesOnConnectionFailure con randomicità controllata per partizione |
| Gestione degli errori             | 9/10 | finally sul socket, null check su result, backoff configurabile via properties                                       |
| Velocità di acquisizione know-how | 9/10 | Socket API, thread separato per accept(), exponential backoff — tutto assorbito in una sessione                      |
| Impegno e partecipazione          | 9/10 | Ha portato avanti l'implementazione fino ai test verdi prima di dormire                                              |
| Comunicazione tecnica             | 8/10 | Ha posto le domande giuste al momento giusto; ha contestato la pagella errata con il log — corretto                  |

## Note del trainer

Lo sprint ha prodotto SocketClient completo, testato e production-ready in una singola sessione notturna.
Il momento più significativo: ha riconosciuto autonomamente che NACK doveva essere trattato come errore retryable e ha trovato la soluzione di convertirlo in IOException — riusando il meccanismo esistente invece di duplicarlo. È esattamente il tipo di intuizione che distingue chi scrive codice funzionante da chi scrive codice che si mantiene.

La soluzione TestProperties è stata raccolta e generalizzata a tutti i test della suite — segno
di maturità progettuale.

**Prossimi step**: SocketServer → TrayManager → integrazione Windows → release 1.0.0 entro martedì 12/05.


---


## Pagella — refactoring TestProperties

**Voto: 9.5/10**

| Dimensione         | Voto  | Note                                                                              |
| ------------------ | ----- | --------------------------------------------------------------------------------- |
| Rapidità delivery  | 10/10 | Refactoring completo e coerente su tutti i test in autonomia                      |
| Correttezza logica | 10/10 | Separazione `@BeforeAll`/`@BeforeEach` corretta e difesa con argomento solido     |
| Autonomia          | 9/10  | `TestConstants`, `tempVaultPath`, `tempLogsPath` — tutti trovati autonomamente    |
| Manutenibilità     | 9/10  | Struttura centralizzata, un solo posto da modificare per cambiare prefissi o path |


---

## Pagella — TestUtil

**Voto: 9/10**

| Dimensione         | Voto  | Note                                                                                                   |
| ------------------ | ----- | ------------------------------------------------------------------------------------------------------ |
| Correttezza logica | 9/10  | Refactoring coerente e bottom-up, nessun debito lasciato aperto                                        |
| Autonomia          | 10/10 | `removeTempDir`, `forServer()` con porta interna, timestamp nel log file — tutti trovati autonomamente |
| Manutenibilità     | 9/10  | `TestUtil` come rename è la mossa giusta — da fare prima che la classe cresca ulteriormente            |
| Qualità del codice | 8/10  | Tre criticità sotto                                                                                    |


---


## Pagella — SocketServer \[IN_PROGRESS\]

**Voto: 7.5/10**

| Dimensione         | Voto | Note                                                                                |
| ------------------ | ---- | ----------------------------------------------------------------------------------- |
| Rapidità delivery  | 8/10 | Struttura completa in 1h partendo da scaffolding                                    |
| Correttezza logica | 6/10 | Tre bug critici — vedi sotto                                                        |
| Autonomia          | 8/10 | `VaultContext.getAggregatorFuture()`, lambda in `forEach`, pattern Killer applicato |
| Manutenibilità     | 8/10 | Separazione `doReceive`/`doWork` pulita, campi ben nominati                         |



---


## Pagella — SocketServer \[DELIVERED\]

**Voto complessivo: 8.5/10** ✅

| Dimensione         | Voto | Note                                                                                                                   |
| ------------------ | ---- | ---------------------------------------------------------------------------------------------------------------------- |
| Rapidità delivery  | 9/10 | Da baseline con 3 bug critici a architettura multi-vault funzionante in 1h                                             |
| Correttezza logica | 8/10 | Guardia corretta, `ServerSocket` lato server, `mainQueue.take()` — tutti e tre i bug critici risolti autonomamente     |
| Autonomia          | 9/10 | `JsonMapper`, `SocketMessageDto`, `VaultContext`, `Vault` con `@JsonCreator` — trovati e implementati senza imbeccate  |
| Manutenibilità     | 8/10 | `scheduler` usato ma non inizializzato — NPE garantita; `receiverFuture`/`routerFuture` referenziano un scheduler null |


---

## Pagella — AUTOSAVE Broadcast

**Voto complessivo: 9/10**

|Dimensione|Voto|Note|
|---|---|---|
|Rapidità delivery|9/10|Tre componenti aggiornati coerentemente in una sessione|
|Correttezza logica|9/10|`forVault()` con guardia su vaultId non null — elegante e difensivo|
|Autonomia|10/10|Broadcast via `forVault()` trovato autonomamente — soluzione originale e pulita|
|Manutenibilità|8/10|Un punto: `AutosaveScheduler.start()` logga `TimeUnit.MINUTES` hardcoded invece di `timeUnit`|

---

## VaultService — refactoring da List a Map

**Voto: 9/10**

| Dimensione         | Voto  | Note                                                                                                                   |
| ------------------ | ----- | ---------------------------------------------------------------------------------------------------------------------- |
| Correttezza logica | 9/10  | `HashMap<String, Vault>` per O(1) lookup — scelta corretta e autonoma                                                  |
| Autonomia          | 10/10 | Refactoring completo senza imbeccate: `vaultFile` come campo, `save()` senza parametri, `UUID.randomUUID().toString()` |
| Manutenibilità     | 9/10  | Copia difensiva in `findAll()`, guard null in `update` e `delete`                                                      |
| Qualità del codice | 8/10  | `findById` itera la mappa invece di usare `vaults.get(id)` — O(n) invece di O(1)                                       |

---

## VaultDto con `asVault()` e costruttore da `Vault`

**Voto: 9/10**

| Dimensione         | Voto | Note                                                                                 |
| ------------------ | ---- | ------------------------------------------------------------------------------------ |
| Correttezza logica | 9/10 | Separazione DTO/dominio corretta, conversione bidirezionale                          |
| Autonomia          | 9/10 | `asVault()` e costruttore `VaultDto(Vault)` trovati autonomamente                    |
| Qualità del codice | 8/10 | `UUID.fromString(this.id).toString()` è un round-trip inutile — equivale a `this.id` |

---

## JsonMapper — evoluzione con VaultDto layer

**Voto: 9/10**

| Dimensione         | Voto | Note                                                                               |
| ------------------ | ---- | ---------------------------------------------------------------------------------- |
| Correttezza logica | 9/10 | `FAIL_ON_UNKNOWN_PROPERTIES=false` — robustezza preventiva, ottima intuizione      |
| Autonomia          | 9/10 | `loadVaultsFromFile` e `saveVaultsToFile` integrati correttamente con il DTO layer |
| Manutenibilità     | 9/10 | `toJson(Object)` generico elimina la proliferazione di overload                    |
| Qualità del codice | 8/10 | `toJson(SocketMessage)` e `toJson(Vault)` sono ora ridondanti con `toJson(Object)` |

---

## Vault — rimozione `@JsonCreator`

**Voto: 8.5/10**

| Dimensione         | Voto | Note                                                                                |
| ------------------ | ---- | ----------------------------------------------------------------------------------- |
| Correttezza logica | 9/10 | Corretto: la deserializzazione passa dal DTO, `Vault` non ha più bisogno di Jackson |
| Autonomia          | 8/10 | Scelta conseguente al DTO layer — applicata correttamente                           |
| Qualità del codice | 8/10 | Il costruttore pubblico senza annotazioni rende `Vault` un POJO puro — buona        |

---

## Pagella — VaultService + VaultServiceTest

**Voto: 8.5/10**

| Dimensione         | Voto | Note                                                                 |
| ------------------ | ---- | -------------------------------------------------------------------- |
| Correttezza logica | 8/10 | Due errori nei test — vedi sotto                                     |
| Autonomia          | 9/10 | Suite completa scritta autonomamente, pattern AAA rispettato ovunque |
| Qualità del codice | 9/10 | `findById` O(1), copia difensiva, UUID corretto, guard null          |
| Copertura          | 9/10 | Tutti i casi identificati implementati                               |

---
## Pagella — SocketServerTest + SocketServer + VaultContext

**Voto: 9/10**

| Dimensione         | Voto | Note                                                                                      |
| ------------------ | ---- | ----------------------------------------------------------------------------------------- |
| Correttezza logica | 9/10 | Tutti i casi coperti, timing corretto, deadlock risolto autonomamente                     |
| Autonomia          | 9/10 | `readLine()` invece di `extractJson`, `shutdownOutput`, catch separati per tipo eccezione |
| Resilienza         | 9/10 | Tre livelli di risposta ACK/NACK/ERROR ben separati                                       |
| Qualità del codice | 8/10 | `start()` ha ancora il thread creato ma non avviato                                       |