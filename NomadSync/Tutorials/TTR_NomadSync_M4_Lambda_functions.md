# Lambda Functions

## Cos'è una lambda

Una lambda è un blocco di codice anonimo — una funzione senza nome che può essere passata come argomento, assegnata a una variabile, o usata inline.

Introdotte in Java 8. Sintassi: `(parametri) -> corpo`

---

## Sintassi base

```java
// funzione senza parametri
Runnable r = () -> System.out.println("Hello");

// un parametro (parentesi opzionali)
Consumer<String> print = s -> System.out.println(s);

// più parametri
Comparator<Integer> cmp = (a, b) -> a - b;

// corpo multi-riga
Runnable r = () -> {
    System.out.println("riga 1");
    System.out.println("riga 2");
};
```

---

## Interfacce funzionali

Una lambda può sostituire qualsiasi **interfaccia funzionale** — un'interfaccia con un solo metodo astratto (`@FunctionalInterface`).

|Interfaccia|Firma|Uso tipico|
|---|---|---|
|`Runnable`|`() -> void`|thread, task|
|`Callable<T>`|`() -> T`|task con risultato|
|`Consumer<T>`|`T -> void`|forEach, logging|
|`Supplier<T>`|`() -> T`|factory, lazy init|
|`Function<T,R>`|`T -> R`|trasformazione|
|`Predicate<T>`|`T -> boolean`|filtro|
|`Comparator<T>`|`(T,T) -> int`|ordinamento|

---

## Uso con le collezioni — Stream API

```java
List<String> names = List.of("Alice", "Bob", "Charlie");

// forEach con Consumer
names.forEach(name -> System.out.println(name));

// filter con Predicate
names.stream()
     .filter(name -> name.startsWith("A"))
     .forEach(System.out::println);

// map con Function
List<Integer> lengths = names.stream()
     .map(name -> name.length())
     .toList();

// sorted con Comparator
names.stream()
     .sorted((a, b) -> a.compareTo(b))
     .toList();
```

---

## Method references — forma abbreviata

Quando la lambda chiama solo un metodo esistente, si può usare `::`:

```java
// equivalenti
names.forEach(name -> System.out.println(name));
names.forEach(System.out::println);

// metodo di istanza
names.forEach(String::toUpperCase);

// metodo statico
List.of("1","2","3").stream()
    .map(Integer::parseInt)
    .toList();

// costruttore
Supplier<ArrayList<String>> factory = ArrayList::new;
```

---

## Lambdas e thread

```java
// Runnable con lambda — usato in ObsidianSync
Thread t = new Thread(() -> {
    while (!Thread.currentThread().isInterrupted()) {
        // lavoro del thread
    }
});
t.start();

// riferimento a metodo di istanza
Thread t = new Thread(this::doWork);
```

---

## Cattura di variabili — effectively final

Una lambda può usare variabili del contesto esterno solo se sono **effectively final** — non riassegnate dopo l'inizializzazione.

```java
int port = 4242;           // effectively final — ok
String host = "localhost"; // effectively final — ok

Runnable r = () -> System.out.println(host + ":" + port);

// ERRORE: port viene riassegnato dopo la lambda
port = 8080;               // compile error: variable used in lambda must be final
```

---

## forEach con Map — usato in ObsidianSync

```java
Map<String, VaultContext> vaults = new HashMap<>();

// itera su tutti i vault e cancella i loro futures
vaults.values().forEach(vault -> vault.getAggregatorFuture().cancel(true));

// itera su chiave e valore
vaults.forEach((vaultId, context) -> {
    System.out.println(vaultId + " → " + context.getQueue().size());
});
```

---

## Comparator con lambda — PriorityBlockingQueue

```java
// ordine naturale (usa Comparable)
PriorityBlockingQueue<SyncEvent> queue = new PriorityBlockingQueue<>();

// ordine custom con lambda
PriorityBlockingQueue<SyncEvent> queue = new PriorityBlockingQueue<>(
    11,
    (a, b) -> Integer.compare(a.getPriority(), b.getPriority())
);

// equivalente con method reference
PriorityBlockingQueue<SyncEvent> queue = new PriorityBlockingQueue<>(
    11,
    Comparator.comparingInt(SyncEvent::getPriority)
);
```

---

## Cheat-sheet

```java
() -> expr                    // nessun parametro
x -> expr                     // un parametro
(x, y) -> expr                // due parametri
(x, y) -> { stmt; return v; } // corpo multi-riga

obj::method                   // metodo di istanza su oggetto fisso
Type::method                  // metodo di istanza su tipo
Type::staticMethod            // metodo statico
Type::new                     // costruttore
```
