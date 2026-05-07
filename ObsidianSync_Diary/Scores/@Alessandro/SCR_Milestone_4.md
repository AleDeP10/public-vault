# Pagelle ottenute durante Milestone 3

---

### Pagella — aggiornamento test

**Voto: 9.5/10**

| Dimensione         | Voto  | Note                                                                                                                                                                                          |
| ------------------ | ----- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Rapidità delivery  | 10/10 | Suite aggiornata autonomamente, tutti e 4 i nuovi casi identificati e implementati                                                                                                            |
| Correttezza logica | 9/10  | Copertura completa: dirty/clean × commit/no commit × PUSH_LOGOFF/PUSH_MANUAL                                                                                                                  |
| Autonomia          | 10/10 | Ha recepito il feedback sui casi mancanti e li ha aggiunti senza imbeccate                                                                                                                    |
| Manutenibilità     | 9/10  | Un punto migliorabile: `pushManual_withChanges_commitsAndPushes` e `pushLogoff_withChanges_commitsAndPushes` sono identici — candidati a parametrizzazione con `@ParameterizedTest` in futuro |

# Pagella — refactor eccezioni

**Voto complessivo: 8.5/10**

|File|Voto|Note principali|
|---|---|---|
|`CommandUtil`|8/10|`NETWORK_PATTERNS` non `final`, pattern matching su `e.getMessage()` potrebbe NPE se il messaggio è null|
|`GitService`|8/10|Catch `NetworkException` silenzioso su operazioni locali — la domanda nei commenti è giusta, risposta sotto|
|`GitServiceTest`|9/10|Signature corrette, isolamento mantenuto, nessun test broken|
|`SyncOrchestrator`|8.5/10|Domanda nei commenti giusta — risposta sotto|

# Pagella — Introduzione PULL_MANUAL

**Voto complessivo: 8.7/10**

|Dimensione|Voto|Note|
|---|---|---|
|Rapidità delivery|9/10|`PULL_MANUAL` implementato in autonomia in meno di 30 minuti come stimato|
|Correttezza logica|9/10|Case unificato con `PULL_LOGON` corretto; priorità 1 corretta|
|Autonomia|9/10|Ha identificato il caso d'uso reale, proposto la soluzione e implementato senza imbeccate|
|Qualità del codice|8/10|`vaultPath` nei metodi anticipa il multi-vault — scelta lungimirante ma ahead of schedule|
|Visione architetturale|9/10|La proposta tray con icona refresh per vault è coerente e ben motivata|


---


# SCR — Sprint SocketClient

## Punteggio complessivo: 8.8 / 10

| Dimensione | Voto | Note |
|---|---|---|
| Qualità del codice prodotto | 9/10 | NACK→IOException è una soluzione elegante che riusa il meccanismo esistente senza duplicare logica |
| Comprensione dei requisiti | 9/10 | Ha anticipato PULL_MANUAL e il caso d'uso multi-sessione senza che venisse richiesto |
| Autonomia tecnica | 9/10 | ServerSocket(0), Runnable pluggabile, TestProperties — tutti trovati autonomamente |
| Testing | 8/10 | Pattern serverBehavior + startServer() pulito; retriesOnConnectionFailure con randomicità controllata per partizione |
| Gestione degli errori | 9/10 | finally sul socket, null check su result, backoff configurabile via properties |
| Velocità di acquisizione know-how | 9/10 | Socket API, thread separato per accept(), exponential backoff — tutto assorbito in una sessione |
| Impegno e partecipazione | 9/10 | Ha portato avanti l'implementazione fino ai test verdi prima di dormire |
| Comunicazione tecnica | 8/10 | Ha posto le domande giuste al momento giusto; ha contestato la pagella errata con il log — corretto |

## Note del trainer

Lo sprint ha prodotto SocketClient completo, testato e production-ready in una singola sessione notturna.
Il momento più significativo: ha riconosciuto autonomamente che NACK doveva essere trattato come errore retryable e ha trovato la soluzione di convertirlo in IOException — riusando il meccanismo esistente invece di duplicarlo. È esattamente il tipo di intuizione che distingue chi scrive codice funzionante da chi scrive codice che si mantiene.

La soluzione TestProperties è stata raccolta e generalizzata a tutti i test della suite — segno
di maturità progettuale.

**Prossimi step**: SocketServer → TrayManager → integrazione Windows → release 1.0.0 entro martedì 12/05.