# AssertJ

## Cos'è

Libreria di asserzioni fluente per Java — alternativa alle asserzioni di JUnit 5 (`assertEquals`, `assertTrue` ecc.). La differenza è nella leggibilità: AssertJ usa una catena di metodi che si legge quasi come una frase in inglese.

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

## Asserzioni base

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

## Asserzioni su collezioni

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

## Soft assertions — tutte le asserzioni anche se una fallisce

```java
// con asserzioni normali, il test si ferma al primo fallimento
// con SoftAssertions, raccoglie tutti i fallimenti e li riporta insieme
SoftAssertions softly = new SoftAssertions();
softly.assertThat(event.getRetryCount()).isEqualTo(1);
softly.assertThat(event.getRetryDelay()).isEqualTo(60_000L);
softly.assertThat(event.getType()).isEqualTo(EventType.PULL_LOGON);
softly.assertAll(); // lancia tutti i fallimenti raccolti
```

## Perché preferire AssertJ a JUnit 5 assertions

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

```java
assertThat(event)
    .extracting(SyncEvent::getRetryCount, SyncEvent::getRetryDelay)
    .containsExactly(1, 60_000L);
```


---


## Secondo argomento di `verify()`

Il secondo argomento è un `VerificationMode` — definisce quante volte ti aspetti che il metodo sia stato chiamato.

| Modalità        | Significato                               |
| --------------- | ----------------------------------------- |
| `times(n)`      | esattamente n volte                       |
| `never()`       | mai — equivalente a `times(0)`            |
| `atLeast(n)`    | almeno n volte                            |
| `atMost(n)`     | al massimo n volte                        |
| `atLeastOnce()` | almeno una volta                          |
| `only()`        | questa e solo questa interazione sul mock |

`verify(mock).metodo()` senza secondo argomento è equivalente a `verify(mock, times(1)).metodo()` — esattamente una volta.