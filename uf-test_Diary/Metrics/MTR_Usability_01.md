# Metriche di qualità — uf-test

## Attributi core (tuoi)

**Appetibilità** — il framework suscita curiosità e desiderio di adozione al primo contatto? Si misura sulla chiarezza del README, sulla narrativa del problema che risolve, sull'effetto WOW alla prima demo.

**Semplicità d'uso** — quanto velocemente un utente jqwik esistente riesce a scrivere il primo test con uf-test? Target: meno di 15 minuti dalla dipendenza Maven al primo test verde.

**Unicità e valore aggiunto** — quanto è percepibile la differenza rispetto a fare la stessa cosa a mano con jqwik puro? Si misura sulla diagnostica post-fallimento: uf-test deve spiegare _perché_ un input ha fallito, non solo _che_ ha fallito.

**Integrabilità con jqwik** — uf-test deve funzionare _dentro_ un test jqwik senza frizione. Nessuna configurazione extra, nessun runner alternativo. Si aggancia come `Arbitrary` custom e sparisce nel workflow esistente.

**Curva di apprendimento** — un developer con background ASD capisce il modello mentale in meno di 30 minuti. Un developer senza background ASD ci arriva comunque, con documentazione adeguata, in meno di 2 ore.

---

## Attributi proposti — layer usabilità

**Leggibilità diagnostica** — quando un test fallisce, il report deve essere umano, non tecnico. Non un dump della struttura UF ma una frase: _"l'input 7 appartiene alla partizione [SUCCESS] ma ha prodotto FAILURE — possibile regressione sul boundary X."_ Questo è il differenziatore più forte sul mercato.

**Reversibilità della diagnostica** — dato un fallimento, l'utente può risalire alla partizione di origine, alla storia delle union che l'ha generata, e al rappresentante della classe. Navigazione bidirezionale sul grafo come strumento di debug, non solo di generazione.

**Componibilità delle partizioni** — le partizioni devono potersi comporre tra loro per testare funzioni multi-parametro. `PartitionOf(int) x PartitionOf(String)` deve produrre una partizione del prodotto cartesiano con outcome combinato. Senza questo, uf-test scala male su metodi reali.

**Estensibilità del tipo** — la struttura UF non deve essere vincolata a tipi primitivi. Qualsiasi tipo con una semantica di equivalenza definita dall'utente deve poter diventare elemento di una partizione. Si misura: quante righe servono per registrare un tipo custom?

**Fail-fast sul misconfiguration** — se le partizioni non coprono l'universo di input dichiarato, uf-test deve lanciare un errore esplicito in fase di setup, non un falso positivo silenzioso. Questo attacca direttamente il rischio principale identificato in precedenza.

---

## Scorecard corrente

|Metrica|Stato attuale|Priorità|
|---|---|---|
|Appetibilità|concept solido, narrativa da costruire|alta|
|Semplicità d'uso|non ancora misurabile|alta|
|Unicità e valore aggiunto|genuina, layer diagnostico differenziante|alta|
|Integrabilità jqwik|fattibile, da prototipare|alta|
|Curva di apprendimento|dipende dalla documentazione|media|
|Leggibilità diagnostica|non implementata|alta|
|Reversibilità diagnostica|core dell'idea, da progettare|alta|
|Componibilità partizioni|non ancora considerata|media|
|Estensibilità del tipo|generics Java, fattibile|media|
|Fail-fast misconfiguration|non ancora considerata|alta|

**Possibilità aggiornata di avvicinarsi al 10**: se leggibilità diagnostica e reversibilità vengono implementate bene, il delta da 5 a 7-8 è realistico. Il layer diagnostico è il vero differenziatore — jqwik trova il controesempio, uf-test lo _spiega_.