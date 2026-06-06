# GRM — Milestone 3: Testing

---

## DTR

---

### [M3] Stack di testing: JUnit 5 + Mockito + AssertJ + Awaitility

**Contesto**: il progetto non dispone di test automatizzati. Prima di procedere all'integrazione con Windows, è necessario coprire i componenti core con test unitari. Lo stack deve essere moderno, adottato in ambito enterprise, e coerente con l'obiettivo di migrazione ai microservizi.

**Decisione**: JUnit 5 (Jupiter) come test runner, Mockito per i mock, AssertJ per le asserzioni fluenti, Awaitility per la sincronizzazione dei test asincroni.

**Motivazione**: combinazione standard nei progetti Spring Boot e Java enterprise moderni. JUnit 5 è lo standard de facto; Mockito è il companion naturale per l'isolamento delle dipendenze; AssertJ produce asserzioni leggibili ed errori descrittivi; Awaitility elimina una classe intera di bug nei test asincroni senza richiedere `CountDownLatch` manuale.

**Alternative scartate**:

- `CountDownLatch` nativo per il testing asincrono — corretto ma verboso, soggetto a bug di timing
- JUnit 4 — legacy, superato dalla riscrittura JUnit 5

**Dipendenze Maven**:

```xml
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>5.10.2</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-core</artifactId>
    <version>5.11.0</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.assertj</groupId>
    <artifactId>assertj-core</artifactId>
    <version>3.25.3</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.awaitility</groupId>
    <artifactId>awaitility</artifactId>
    <version>4.2.1</version>
    <scope>test</scope>
</dependency>
```

---

### [M3] Strategia di testing: unit test con mock delle dipendenze esterne

**Contesto**: `SyncOrchestrator` dipende da `GitService`, `LogService`, `NotificationHook`. Eseguire comandi Git reali nei test introduce dipendenze dalla rete, dallo stato del repository e dal filesystem.

**Decisione**: tutti i test unitari mockano le dipendenze tramite Mockito. `GitService` non esegue mai comandi reali durante i test.

**Motivazione**: test isolati, deterministici, veloci. Il pattern rispecchia l'architettura a dependency injection già adottata nell'orchestratore.

---

### [M3] Sincronizzazione test asincroni tramite Awaitility

**Contesto**: `SyncOrchestrator` consuma eventi su un worker thread separato. I test devono attendere che l'evento sia stato processato prima di eseguire le `verify` di Mockito. Un'attesa fissa (`Thread.sleep`) è fragile e dipendente dalla velocità della macchina.

**Decisione**: utilizzare Awaitility con polling attivo e timeout esplicito.

**Motivazione**: Awaitility è lo standard per il testing asincrono in Java. Elimina il problema del timing variabile senza introdurre sleep arbitrari. Sintassi fluente, coerente con AssertJ.

**Pattern di riferimento**:

```java
await().atMost(2, SECONDS).untilAsserted(() ->
    verify(gitService).pull()
);
```

---

### [M3] Separazione eccezioni: NetworkException vs GitException in GitService

**Contesto**: il retry con exponential backoff deve applicarsi solo ai fallimenti di rete, non agli errori locali Git (conflitti, stash vuoto, repo corrotto). GitService deve distinguere i due casi nel tipo di eccezione lanciata.

**Decisione**: `GitService` lancia `NetworkException` per errori di connettività e `GitException` per errori locali. `SyncOrchestrator` gestisce i due casi con catch separati.

**Motivazione**: semantica chiara, retry applicato solo dove ha senso. Pattern standard nei sistemi distribuiti.

---

## Obiettivi dello sprint — cosa entra nel done

| #   | Obiettivo                                   | Criterio di accettazione                                                                                               |
| --- | ------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| 1   | Dipendenze Maven configurate                | `mvn test` compila ed esegue senza errori                                                                              |
| 2   | `GitServiceTest`                            | Copertura dei metodi pubblici: stash, pull, stashPop, commitLocal, push, hasChanges                                    |
| 3   | `SyncEventQueueTest`                        | Verifica deduplicazione latest-wins, verifica ordinamento per priorità                                                 |
| 4   | `SyncOrchestratorTest` — flusso happy path  | PULL_LOGON esegue stash→pull→stashPop in ordine; AUTOSAVE committa locale senza push                                   |
| 5   | `SyncOrchestratorTest` — flusso errore rete | NetworkException su pull scatena 3 retry con delay crescente; dopo il terzo fallimento viene chiamato notificationHook |
| 6   | `SyncOrchestratorTest` — deduplicazione     | Due AUTOSAVE in rapida successione producono una sola esecuzione                                                       |
| 7   | `SyncOrchestratorTest` — guard hasChanges   | AUTOSAVE e PUSH senza modifiche non chiamano mai commitLocal                                                           |

---

## Rischi identificati — cosa potrebbe bloccare

|Rischio|Probabilità|Impatto|Mitigazione|
|---|---|---|---|
|Awaitility timeout instabile su macchine lente|media|medio|impostare timeout generosi (2-5s) nei test asincroni; evitare timeout < 1s|
|`PriorityBlockingQueue` non supporta rimozione selettiva atomica per latest-wins|alta|alto|wrappare la coda in un blocco `synchronized` per la fase publish; verificare comportamento con test dedicato|
|Worker loop non termina nei test — porta a leak di thread|media|medio|esporre metodo `shutdown()` sull'orchestratore; chiamarlo in `@AfterEach`|
|Mockito non intercetta metodi `final` per default|bassa|medio|se GitService ha metodi final, aggiungere `mockito-extensions` o rimuovere il modificatore|
|Verifica dell'ordine delle chiamate fallisce per race condition|media|alto|usare `InOrder` di Mockito combinato con Awaitility per garantire che il thread abbia completato prima della verifica|

---

## Know-how da acquisire — cosa studiare prima di scrivere

|Area|Concetto|Perché serve|
|---|---|---|
|JUnit 5|`@BeforeEach`, `@AfterEach`, `@Test`|lifecycle base dei test|
|JUnit 5|`@ExtendWith(MockitoExtension.class)`|alternativa moderna a `MockitoAnnotations.openMocks()`|
|Mockito|`@Mock`, `when().thenReturn()`, `doThrow().when()`|definire comportamento dei mock|
|Mockito|`verify()`, `times()`, `never()`, `InOrder`|asserire le interazioni|
|Mockito|`any()`, `anyString()`|matcher per argomenti non deterministici|
|AssertJ|`assertThat().isEqualTo()`, `isTrue()`, `isInstanceOf()`|asserzioni fluenti|
|Awaitility|`await().atMost().untilAsserted()`|sincronizzazione test asincroni|
|Java Concurrency|`Thread.interrupt()`, shutdown pattern|terminazione pulita del worker loop nei test|