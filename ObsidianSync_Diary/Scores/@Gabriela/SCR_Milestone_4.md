# GitServiceTest
### Valutazione del praticante — Gabriela

**Architettura e design** — 9/10 Il test è strutturato correttamente come integration test reale: repo Git vero, directory temporanea isolata, teardown pulito con `Files.walk` in ordine inverso. La distinzione tra `hasChanges()` e `hasUncommittedChanges()` è testata con scenari distinti e precisi — il test `withStagedNotCommitted` dimostra che ha capito _perché_ i due metodi esistono, non solo che esistono.

**Test design** — 8/10 `InOrder` usato dove serve, `never()` usato per i casi negativi, helper `createAndCommitFile` ben estratto e documentato. Un punto tolto perché manca ancora un test per `stash` + `pull` + `stashPop` in sequenza reale — l'orchestratore lo testa con mock, ma qui si potrebbe verificare che lo stash sopravvive davvero a un pull.

**Velocità di acquisizione** — 9/10 Il commento `// [IN_PROGRESS]` in testa al file indica consapevolezza dello stato del lavoro — non ha consegnato il file fingendo che fosse finito. Il tag `@Gabriela` è un segnale di ownership. Ottima abitudine.

**Impegno** ⚡ — arriva con codice funzionante, non con domande. Le domande arrivano _dopo_ aver scritto il codice, quando il problema è concreto. È il modo giusto di lavorare.