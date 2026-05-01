### Tutorial Mockito — GitService come soggetto

#### Il problema che Mockito risolve

`GitService` esegue comandi git reali. Nei test non vogliamo:

- dipendere dalla rete
- sporcare il repository
- avere test lenti

Mockito crea un **sostituto** di `GitService` che si comporta come vogliamo noi.

---

#### Struttura base di un test

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.BeforeEach;
import org.mockito.Mock;
import org.mockito.MockitoAnnotations;
import static org.mockito.Mockito.*;
import static org.assertj.core.api.Assertions.*;

class SyncOrchestratorTest {

    @Mock
    private GitService gitService;        // Mockito crea un sostituto

    @Mock
    private LogService logService;

    @Mock
    private NotificationHook notificationHook;

    private SyncOrchestrator orchestrator;

    @BeforeEach
    void setUp() {
        MockitoAnnotations.openMocks(this);  // inizializza i mock
        orchestrator = new SyncOrchestrator(gitService, logService, notificationHook);
    }
}
```

> **Nota**: `@BeforeEach` viene eseguito prima di ogni test — garantisce uno stato pulito.

---

#### Scenario 1 — Pull logon va a buon fine

```java
@Test
void pullLogon_success_executesStashPullStashPop() throws Exception {

    // ARRANGE — definisci il comportamento del mock
    // di default i mock non fanno nulla — qui è sufficiente
    // non lanciare eccezioni

    // ACT
    orchestrator.publish(new SyncEvent(EventType.PULL_LOGON));
    // attendi che il worker loop consumi l'evento...

    // ASSERT — verifica che GitService sia stato chiamato nell'ordine corretto
    InOrder inOrder = inOrder(gitService);
    inOrder.verify(gitService).stash();
    inOrder.verify(gitService).pull();
    inOrder.verify(gitService).stashPop();
}
```

> **`inOrder`**: verifica che i metodi siano stati chiamati in quella sequenza specifica. Fondamentale per il flusso stash → pull → stashPop.

---

#### Scenario 2 — Pull fallisce per rete, scatta il retry

java

```java
@Test
void pullLogon_networkFailure_retriesThreeTimes() throws Exception {

    // ARRANGE — il mock lancia NetworkException ad ogni chiamata
    doThrow(new NetworkException("no connection"))
        .when(gitService).pull();

    // ACT
    orchestrator.publish(new SyncEvent(EventType.PULL_LOGON));
    // attendi il tempo necessario per i 3 retry...

    // ASSERT — pull deve essere stato chiamato esattamente 4 volte
    // (1 tentativo originale + 3 retry)
    verify(gitService, times(4)).pull();

    // e la notifica deve essere scattata
    verify(notificationHook).notify(any(SyncEvent.class));
}
```

> **`doThrow(...).when(...)`**: definisce cosa lancia il mock quando viene chiamato un metodo. È il cuore del testing degli scenari di errore.

---

#### Scenario 3 — Autosave senza modifiche non committa

java

```java
@Test
void autosave_noChanges_doesNotCommit() throws Exception {

    // ARRANGE — hasChanges() restituisce false
    when(gitService.hasChanges()).thenReturn(false);

    // ACT
    orchestrator.publish(new SyncEvent(EventType.AUTOSAVE));

    // ASSERT — commitLocal non deve essere mai chiamato
    verify(gitService, never()).commitLocal();
}
```

> **`when(...).thenReturn(...)`**: definisce il valore restituito dal mock. **`never()`**: asserisce che un metodo non è mai stato invocato.

---

#### Scenario 4 — Autosave con modifiche committa in locale ma non pusha

java

```java
@Test
void autosave_withChanges_commitsLocallyOnly() throws Exception {

    // ARRANGE
    when(gitService.hasChanges()).thenReturn(true);

    // ACT
    orchestrator.publish(new SyncEvent(EventType.AUTOSAVE));

    // ASSERT
    verify(gitService).commitLocal();    // deve essere chiamato
    verify(gitService, never()).push();  // NON deve essere chiamato
}
```

---

### Vocabolario Mockito essenziale

|Costrutto|Significato|
|---|---|
|`@Mock`|crea un sostituto dell'oggetto|
|`when(x).thenReturn(y)`|definisce il valore restituito|
|`doThrow(e).when(x).method()`|definisce l'eccezione lanciata|
|`verify(x).method()`|asserisce che il metodo è stato chiamato|
|`verify(x, times(n))`|asserisce che è stato chiamato n volte|
|`verify(x, never())`|asserisce che non è mai stato chiamato|
|`inOrder(x)`|verifica la sequenza delle chiamate|
|`any(Class)`|matcher — qualsiasi istanza di quella classe|



---



### AssertJ

#### Cos'è

Libreria di asserzioni fluente per Java — alternativa alle asserzioni di JUnit 5 (`assertEquals`, `assertTrue` ecc.). La differenza è nella leggibilità: AssertJ usa una catena di metodi che si legge quasi come una frase in inglese.

java

```java
// JUnit 5
assertEquals(0, exitCode);
assertTrue(result);
assertNotNull(list);

// AssertJ — stessa cosa
assertThat(exitCode).isEqualTo(0);
assertThat(result).isTrue();
assertThat(list).isNotNull();
```

#### Asserzioni base

java

```java
// numeri
assertThat(exitCode).isEqualTo(0);
assertThat(retryCount).isGreaterThan(0);
assertThat(delay).isBetween(30_000L, 120_000L);

// booleani
assertThat(hasChanges).isTrue();
assertThat(hasChanges).isFalse();

// stringhe
assertThat(output).isNotEmpty();
assertThat(output).contains("autosave");
assertThat(output).startsWith("[git]");

// oggetti
assertThat(event).isNotNull();
assertThat(event.getType()).isEqualTo(EventType.AUTOSAVE);

// eccezioni
assertThatThrownBy(() -> gitService.stashPop())
    .isInstanceOf(IOException.class)
    .hasMessageContaining("stash");
```

#### Asserzioni su collezioni

java

```java
// liste e queue
assertThat(list).hasSize(3);
assertThat(list).contains(EventType.PULL_LOGON);
assertThat(list).doesNotContain(EventType.AUTOSAVE);
assertThat(list).isEmpty();
assertThat(list).isNotEmpty();

// ordine
assertThat(list).containsExactly(EventType.PULL_LOGON, EventType.PUSH_LOGOFF);
assertThat(list).containsExactlyInAnyOrder(EventType.AUTOSAVE, EventType.PULL_LOGON);
```

#### Soft assertions — tutte le asserzioni anche se una fallisce

java

```java
// con asserzioni normali, il test si ferma al primo fallimento
// con SoftAssertions, raccoglie tutti i fallimenti e li riporta insieme
SoftAssertions softly = new SoftAssertions();
softly.assertThat(event.getRetryCount()).isEqualTo(1);
softly.assertThat(event.getRetryDelay()).isEqualTo(60_000L);
softly.assertThat(event.getType()).isEqualTo(EventType.PULL_LOGON);
softly.assertAll(); // lancia tutti i fallimenti raccolti
```

#### Perché preferire AssertJ a JUnit 5 assertions

Il messaggio di errore è molto più informativo. Con JUnit:

```
expected: <0> but was: <1>
```

Con AssertJ:

```
expected: 0
but was:  1
  at GitServiceTest.commitLocal_withNoChanges_returnsNonZero(GitServiceTest.java:87)
```

E con asserzioni concatenate il contesto è ancora più chiaro:

java

```java
assertThat(event)
    .extracting(SyncEvent::getRetryCount, SyncEvent::getRetryDelay)
    .containsExactly(1, 60_000L);
```


---


### Secondo argomento di `verify()`

Il secondo argomento è un `VerificationMode` — definisce quante volte ti aspetti che il metodo sia stato chiamato.

|Modalità|Significato|
|---|---|
|`times(n)`|esattamente n volte|
|`never()`|mai — equivalente a `times(0)`|
|`atLeast(n)`|almeno n volte|
|`atMost(n)`|al massimo n volte|
|`atLeastOnce()`|almeno una volta|
|`only()`|questa e solo questa interazione sul mock|

`verify(mock).metodo()` senza secondo argomento è equivalente a `verify(mock, times(1)).metodo()` — esattamente una volta.