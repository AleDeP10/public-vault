# ObsidianSync — Report Milestone 3

## Obiettivi Raggiunti

- Suite di unit test completa: 32 test su 6 classi, tutti verdi
- `GitServiceTest`: 9 test con repo git temporanea reale — nessun push remoto
- `SyncOrchestratorTest`: 6 test con mock di GitService e NotificationHook
- `SyncEventQueueTest`: 5 test su deduplicazione latest-wins e ordinamento priorità
- `SyncEventTest`: 5 test su compareTo, incrementRetry e toString
- `LogServiceTest`: 5 test su filtraggio livelli, formato timestamp e modalità append
- `AutosaveSchedulerTest`: 2 test con costruttore package-private a millisecondo
- Introdotto `CommandUtil` — utility per esecuzione processi, elimina duplicazione tra GitService e test
- Introdotto costruttore package-private su `SyncEvent` per timestamp controllato nei test
- Introdotto costruttore package-private su `AutosaveScheduler` per TimeUnit configurabile nei test
- Risolto problema compilazione `Process` non `AutoCloseable` su Java 21
- Risolto problema JDK 25 incompatibile con Mockito/ByteBuddy — downgrade a Java 21
- Risolto problema compact source file causato da `Main.java` duplicato fuori dal package

---

## Problematiche Affrontate

|Problema|Risoluzione|
|---|---|
|`mvn test` restituisce "No tests to run"|Cartella `java/` mancante sotto `src/test/` — Maven non riconosceva il source root|
|Mockito non riesce a mockare classi con Java 25|Downgrade JDK a Java 21 — ByteBuddy non supportava Java 25|
|`orchestrator` null nel test nonostante `@BeforeEach`|Il `catch` nel setup nascondeva il fallimento del mock — rimosso il catch, propagata l'eccezione|
|`SyncEventQueue` non mockabile|Non va mockato — è logica pura, va usata istanza reale|
|`@Mock static LogService` non supportato da Mockito|`@Mock` è per istanza, non per classe — rimosso, usata istanza reale|
|`openMocks(this)` in try-with-resources chiudeva il contesto prima del test|Spostato `AutoCloseable` come campo, chiuso in `@AfterEach`|
|`echo` e `>` non funzionano via `ProcessBuilder`|Sono comandi shell — sostituiti con `Files.writeString()`|
|`tearDown` usava `Remove-Item` via `ProcessBuilder`|Comando PowerShell non disponibile come eseguibile — sostituito con `Files.walk()`|
|`git.executable` mancante nelle properties del test|Aggiunto `git.executable=git` nel setup di `GitServiceTest`|
|`stash pop` su coda vuota in test drain|Ciclo di drain corretto: `while (queue.size() > 0)` invece di `while (event == null)`|

---

## Cheat-Sheet Acquisiti

### JUnit 5

- `@BeforeAll` — eseguito una volta per classe, campo `static`
- `@BeforeEach` / `@AfterEach` — eseguiti per ogni test, garantiscono isolamento
- Test senza `@Test` non vengono eseguiti — errore silenzioso
- Le eccezioni nel `@BeforeEach` fanno fallire il test con messaggio chiaro — non catturarle mai silenziosamente

### Mockito

- `mock(Class.class)` — crea un mock della classe
- `when(mock.metodo()).thenReturn(valore)` — configura il comportamento
- `verify(mock).metodo()` — verifica che il metodo sia stato chiamato esattamente una volta
- `verify(mock, never()).metodo()` — verifica che il metodo non sia mai stato chiamato
- `verify(mock, times(n)).metodo()` — verifica n invocazioni
- `InOrder inOrder = inOrder(mock)` — verifica l'ordine delle chiamate
- `inOrder.verify(mock).metodo()` — verifica sequenziale
- `never()` non va usato dentro `InOrder` — usare `verify(mock, never())` standalone
- `anyString()` — matcher per qualsiasi stringa in `verify` e `when`
- `openMocks(this)` restituisce `AutoCloseable` da chiudere in `@AfterEach`
- Mockito non supporta `@Mock` su campi `static`
- ByteBuddy (engine di Mockito) non supporta Java 25 con versioni precedenti alla 5.17

### AssertJ

- `assertThat(x).isEqualTo(y)` — uguaglianza
- `assertThat(x).contains(y)` — stringa contiene sottostringa
- `assertThat(x).doesNotContain(y)` — stringa non contiene sottostringa
- `assertThat(x).containsPattern(regex)` — match con espressione regolare
- `assertThat(x).isGreaterThanOrEqualTo(n)` — confronto numerico
- `assertThat(x).isLessThan(0)` — utile per verificare risultato di `compareTo`
- Preferire `assertThat(x).contains(y)` a `assertThat(x.contains(y)).isTrue()` — messaggi di errore più leggibili

### Testing patterns

- Test con repo git reale: `Files.createTempDirectory()` + `git init` + `Files.walk()` per cleanup
- Costruttore package-private per test: mantiene `final` sui campi, evita setter solo per test
- `Thread.sleep` nei test: usare il minimo indispensabile (1-10ms), non valori arbitrari come 1000ms
- Drain della coda: `while (queue.size() > 0) { queue.consume(); }` non `while (event == null)`
