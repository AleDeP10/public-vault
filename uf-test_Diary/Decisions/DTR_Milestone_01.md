# DTR — uf-test Milestone 1

## Identità del progetto

**Nome**: uf-test **Natura**: framework di testing open source (licenza MIT) basato su Union-Find con Path Compression e Union-By-Rank **Posizionamento**: strumento integrativo a jqwik, non sostitutivo — colma il gap diagnostico post-fallimento che jqwik non copre **Ambito di applicazione**: non vincolato a ObsidianSync — applicabile a tutti i progetti Java presenti e futuri (ToDoList, ecc.) **Obiettivo di lungo periodo**: stabilizzarsi come strumento complementare a jqwik nella stessa misura in cui AssertJ lo è per JUnit 5

---

## Fondamento matematico

Una **partizione** di un insieme S è una famiglia di sottoinsiemi {P₁, P₂, ..., Pₙ} tale che:

- **Copertura**: ogni elemento di S appartiene ad almeno un sottoinsieme
- **Disgiunzione**: nessun elemento appartiene a due sottoinsiemi contemporaneamente
- **Non vuoto**: nessun sottoinsieme è vuoto

**Corrispondenza con Union-Find**:

- `Make-Set(x)` → ogni elemento inizia come partizione singleton, è rappresentante di sé stesso — O(1)
- `Union(x, y)` → due partizioni vengono fuse con un unico rappresentante — le proprietà delle partizioni restano valide
- `Find(x)` → restituisce il rappresentante della classe di equivalenza di x

Le partizioni modellano le **classi di equivalenza funzionale**: tutti gli elementi di una stessa partizione producono il medesimo risultato osservabile (OK o FAILURE) quando passati alla funzione sotto test.

---

## Decisioni architetturali

### [M1-D01] Struttura dati core: grafo bidirezionale invece di albero UF classico

**Contesto**: il solo albero UF consente la navigazione verso il rappresentante ma non la navigazione inversa né la ricostruzione della storia delle fusioni.

**Decisione**: struttura dati basata su grafo bidirezionale che estende UF. Ogni nodo mantiene riferimenti sia al padre (navigazione verso il rappresentante) sia ai figli (navigazione inversa verso tutti i membri della partizione). La storia delle union viene preservata.

**Motivazione**: abilita il layer diagnostico che costituisce il principale differenziatore rispetto a jqwik. Dato un fallimento, l'utente può risalire alla partizione di origine, alla storia delle fusioni e al rappresentante della classe.

**Alternative scartate**: albero UF classico — navigazione monodirezionale, nessuna memoria delle fusioni.

---

### [M1-D02] Ruolo della struttura UF: generatore di input + strumento diagnostico

**Contesto**: il caso d'uso primario non è la sola generazione di input casuali, ma la spiegazione dei fallimenti.

**Decisione**: la struttura espone due responsabilità distinte:

1. **Generazione**: metodo dedicato che restituisce un elemento casuale da una partizione specificata
2. **Diagnostica**: navigazione bidirezionale per risalire dalla partizione al rappresentante e viceversa, con ricostruzione della storia delle union

**Motivazione**: jqwik trova il controesempio; uf-test lo spiega. Questo è il valore aggiunto reale.

---

### [M1-D03] Integrazione con jqwik tramite `Arbitrary` custom

**Contesto**: uf-test deve funzionare dentro un test jqwik senza richiedere configurazione extra o runner alternativi.

**Decisione**: integrazione tramite `Arbitraries.of(collection)` o implementazione di un `Arbitrary` custom che pesca dalla struttura UF. Nessun runner proprietario.

**Motivazione**: abbatte la curva di adozione — un utente jqwik esistente non deve imparare un nuovo ecosistema, solo un nuovo tipo di generatore.

---

### [M1-D04] Fail-fast su misconfiguration

**Contesto**: se le partizioni non coprono correttamente l'universo di input dichiarato, il framework produce falsi positivi silenziosamente — rischio principale identificato.

**Decisione**: validazione esplicita in fase di setup. Se le partizioni sono incomplete o si sovrappongono, uf-test lancia un'eccezione descrittiva prima dell'esecuzione dei test.

**Motivazione**: attacca direttamente il rischio principale. Un framework che non rileva la propria misconfiguration non è affidabile.

---

### [M1-D05] Supporto a tipi generici

**Contesto**: vincolare la struttura a tipi primitivi limiterebbe l'applicabilità a casi reali.

**Decisione**: implementazione con generics Java. Qualsiasi tipo con una semantica di equivalenza definita dall'utente può diventare elemento di una partizione.

**Motivazione**: prerequisito per l'adozione su progetti reali. Si misura in righe di codice necessarie per registrare un tipo custom — target: meno di 5.

---

### [M1-D06] Componibilità delle partizioni per metodi multi-parametro

**Contesto**: metodi reali accettano più parametri. Una struttura che gestisce solo partizioni monodimensionali scala male.

**Decisione**: le partizioni devono essere componibili. `PartitionOf<Integer> × PartitionOf<String>` produce una partizione del prodotto cartesiano con outcome combinato.

**Motivazione**: senza componibilità, uf-test è utilizzabile solo su metodi con un singolo parametro — caso di minoranza nei progetti reali.

**Stato**: requisito identificato, implementazione da progettare.

---

## Requisiti funzionali

|ID|Requisito|Priorità|Stato|
|---|---|---|---|
|RF-01|`Make-Set`, `Union`, `Find` con Path Compression e Union-By-Rank|Alta|Da implementare|
|RF-02|Navigazione bidirezionale (verso rappresentante e verso membri)|Alta|Da progettare|
|RF-03|Estrazione casuale di un elemento da una partizione specificata|Alta|Da implementare|
|RF-04|Integrazione come `Arbitrary` jqwik|Alta|Da prototipare|
|RF-05|Fail-fast su partizioni incomplete o sovrapposte|Alta|Da implementare|
|RF-06|Supporto a generics Java per tipi custom|Media|Da implementare|
|RF-07|Componibilità di partizioni per prodotto cartesiano|Media|Da progettare|
|RF-08|Report diagnostico leggibile in linguaggio naturale post-fallimento|Alta|Da progettare|
|RF-09|Ricostruzione della storia delle union|Media|Da progettare|

---

## Requisiti non funzionali

|ID|Requisito|Target|
|---|---|---|
|RNF-01|Time-to-first-test per utente jqwik esistente|< 15 minuti|
|RNF-02|Linee di codice per registrare un tipo custom|< 5|
|RNF-03|Overhead sul singolo test rispetto a jqwik puro|Trascurabile|
|RNF-04|Licenza|MIT|
|RNF-05|Compatibilità|Java + JUnit 5 + jqwik|

---

## Metriche di qualità — stato iniziale

|Metrica|Stato attuale|Priorità|
|---|---|---|
|Appetibilità|Concept solido, narrativa da costruire|Alta|
|Semplicità d'uso|Non ancora misurabile|Alta|
|Unicità e valore aggiunto|Genuina — layer diagnostico differenziante|Alta|
|Integrabilità con jqwik|Fattibile, da prototipare|Alta|
|Curva di apprendimento|Dipende dalla documentazione|Media|
|Leggibilità diagnostica|Non implementata|Alta|
|Reversibilità diagnostica|Core dell'idea, da progettare|Alta|
|Componibilità delle partizioni|Non ancora considerata|Media|
|Estensibilità del tipo|Generics Java, fattibile|Media|
|Fail-fast misconfiguration|Non ancora considerata|Alta|

---

## Valutazione impatto open source

**Score corrente**: 5/10 come potenziale adozione di massa **Score portfolio**: 9/10 come progetto di punta e differenziatore da recruiter

Il delta tra 5 e 10 dipende da fattori esterni al codice: adozione da parte di developer influenti, visibilità in conferenze JVM, evangelizzazione attiva. Il layer diagnostico — se implementato bene — porta il potenziale a 7-8/10.

---

## Note metodologiche

- Sviluppo incrementale (non Scrum)
- Questa chat funge da Decision Track Record
- Le metriche di usabilità verranno approfondite una per una nei prompt successivi
- Il DTR in inglese per `docs/decisions/` verrà prodotto previa validazione del presente documento