
### jqwik — panoramica

**Cos'è**: libreria Java per Property-Based Testing, si integra con JUnit 5.

**Concetto base**: invece di esempi fissi, definisci _proprietà_ che devono valere per interi domini di input. jqwik genera automaticamente centinaia di casi.

**Dipendenza Maven**:

```xml
<dependency>
    <groupId>net.jqwik</groupId>
    <artifactId>jqwik</artifactId>
    <version>1.8.4</version>
    <scope>test</scope>
</dependency>
```

**Annotazioni chiave**:

| Annotazione           | Significato                                              |
| --------------------- | -------------------------------------------------------- |
| `@Property`           | Sostituisce `@Test` — eseguito N volte con input diversi |
| `@ForAll`             | Parametro generato automaticamente da jqwik              |
| `@IntRange(min, max)` | Vincola il dominio a un intervallo — la tua partizione   |
| `@From("nomeMetodo")` | Usa un `@Provide` custom come generatore                 |

**Esempio con partizione esplicita**:

java

```java
@Provide
Arbitrary<Integer> successAttempts() {
    return Arbitraries.integers().between(2, 3); // partizione B: successo al 2° o 3° tentativo
}

@Property
void socketClient_succeedsAfterPartialRetry(
        @ForAll("successAttempts") int successAt) {
    // successAt è sempre in {2, 3} — partizione garantita
    // comportamento atteso: sempre GREEN
}
```

**Shrinking**: se un caso fallisce, jqwik riduce automaticamente l'input al caso minimo che riproduce il fallimento — equivalente al seed riproducibile ma più intelligente.

**Integrazione con la tua idea UF**: il metodo `@Provide` è il punto di innesto naturale — il tuo generatore UF restituisce un `Arbitrary<T>` pescando dalla partizione corretta. jqwik si occupa dell'esecuzione ripetuta, tu ti occupi della semantica delle partizioni.

