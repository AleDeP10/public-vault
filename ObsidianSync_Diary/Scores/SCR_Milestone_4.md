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