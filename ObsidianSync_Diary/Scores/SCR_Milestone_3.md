# Pagelle ottenute durante Milestone 3

---

### Briciole — valutazione

**Briciola 1** — ragionamento corretto e ben argomentato. L'atomicità dei test è il principio fondamentale: ogni test deve poter girare in isolamento, in qualsiasi ordine, senza dipendere dallo stato lasciato dal precedente. `@BeforeEach/@AfterEach` è la scelta giusta. **9/10**

**Briciola 2** — esatto. `.toFile()` è il ponte tra `Path` (API moderna) e `File` (API legacy). **10/10**


---

## Pagella CommandUtil

**Voto: 8.5/10**

| Dimensione         | Voto | Note                                                                                     |
| ------------------ | ---- | ---------------------------------------------------------------------------------------- |
| Rapidità delivery  | 9/10 | Soluzione identificata e implementata autonomamente                                      |
| Correttezza logica | 9/10 | Refactor pulito, metodi statici appropriati per una utility                              |
| Autonomia          | 9/10 | Naming ragionato, scelta del package `util` corretta                                     |
| Manutenibilità     | 7/10 | `LogService` opzionale non gestito, accoppiamento con LogService in una utility generica |

---


## Pagella GitServiceTest

**Voto: 8/10**

|Dimensione|Voto|Note|
|---|---|---|
|Rapidità delivery|9/10|Suite completa al primo tentativo, tutti i casi coperti|
|Correttezza logica|7/10|Tre problemi tecnici che impedirebbero l'esecuzione reale|
|Autonomia|9/10|Struttura, naming, separazione `@BeforeAll`/`@BeforeEach` — tutto autonomo e corretto|
|Manutenibilità|8/10|`createFile()` helper ben estratto, test leggibili|


---


### Pagella SyncOrchestratorTest

**Voto: 9/10**

| Dimensione         | Voto | Note                                                                                        |
| ------------------ | ---- | ------------------------------------------------------------------------------------------- |
| Rapidità delivery  | 9/10 | Suite completa autonomamente, pattern ARRANGE/ACT/ASSERT applicato con consistenza          |
| Correttezza logica | 9/10 | Un solo bug rilevato (`stashPop` invece di `never().push()`) corretto autonomamente         |
| Autonomia          | 9/10 | `anyString()`, `InOrder`, `never()` — tutti trovati e applicati senza imbeccate             |
| Manutenibilità     | 9/10 | Naming descrittivo, struttura uniforme, commenti pseudocodice mantenuti come documentazione |


---


### Pagella SyncEventQueueTest

**Voto: 8.5/10**

| Dimensione         | Voto | Note                                                                  |
| ------------------ | ---- | --------------------------------------------------------------------- |
| Rapidità delivery  | 9/10 | Suite completa autonomamente, pattern corretto                        |
| Correttezza logica | 8/10 | `Thread.sleep(1000)` funziona ma è fragile — vedi sotto               |
| Autonomia          | 9/10 | Briciola sul timestamp colta e risolta autonomamente con nota critica |
| Manutenibilità     | 8/10 | `setTimestamp` in produzione è un code smell segnalato correttamente  |


---


## Pagella SyncEventTest

**Voto: 9/10**

| Dimensione         | Voto  | Note                                                                    |
| ------------------ | ----- | ----------------------------------------------------------------------- |
| Rapidità delivery  | 10/10 | Suite completa, corretta, autonoma                                      |
| Correttezza logica | 9/10  | Un punto sottile in `compareTo_samePriority_olderPrecedesNewer`         |
| Autonomia          | 9/10  | Costruttore di test usato correttamente, costanti `30_000` riconosciute |
| Manutenibilità     | 9/10  | Pulito, leggibile, nessun import inutile                                |


---


## Pagella LogServiceTest

**Voto: 8.5/10**

| Dimensione         | Voto | Note                                                                                     |
| ------------------ | ---- | ---------------------------------------------------------------------------------------- |
| Rapidità delivery  | 9/10 | Suite quasi completa, pattern helper ben estratti                                        |
| Correttezza logica | 8/10 | `@Test` mancante su `log_atMinLevel_writes`, `Thread.sleep` in `readLogFile` discutibile |
| Autonomia          | 9/10 | Regex per timestamp trovata e applicata correttamente, helper riutilizzabili             |
| Manutenibilità     | 8/10 | `assertThat(logs.contains(...)).isTrue()` — esiste un'API più diretta in AssertJ         |


---


## Pagella `AutosaveSchedulerTest`

**Voto: 8.5/10**

|Dimensione|Voto|Note|
|---|---|---|
|Rapidità delivery|9/10|Briciola colta e applicata autonomamente con refactor pulito|
|Correttezza logica|8/10|Bug nel drain della coda in `stop_preventsSubsequentPublishing`|
|Autonomia|9/10|`TimeUnit`, costruttore package-private, pattern corretto|
|Manutenibilità|8/10|`Assertions.assertNotNull` mischiato con AssertJ — stile inconsistente|