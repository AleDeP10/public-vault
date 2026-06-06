# Tutorial Mockito — GitService come soggetto

#### Il problema che Mockito risolve

`GitService` esegue comandi git reali. Nei test non vogliamo:

- dipendere dalla rete
- sporcare il repository
- avere test lenti

Mockito crea un **sostituto** di `GitService` che si comporta come vogliamo noi.

---

## Struttura base di un test

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

## Scenario 1 — Pull logon va a buon fine

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

## Scenario 2 — Pull fallisce per rete, scatta il retry

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

## Scenario 3 — Autosave senza modifiche non committa

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

## Scenario 4 — Autosave con modifiche committa in locale ma non pusha

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

## Vocabolario Mockito essenziale

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
