# Java Local Sockets
## Cos'è un socket

Un socket è un canale di comunicazione bidirezionale tra due processi. In ObsidianSync viene usato per far comunicare i task di Task Scheduler (client effimeri) con il processo tray persistente (server).

---

## Le due classi fondamentali

|Classe|Ruolo|
|---|---|
|`ServerSocket`|Lato server — aspetta connessioni in entrata|
|`Socket`|Lato client — si connette al server; lato server — rappresenta la connessione accettata|

---

## Lato server — ciclo di vita

```java
// 1. Apre il server su una porta
ServerSocket serverSocket = new ServerSocket(4242);

// 2. Aspetta una connessione — BLOCCANTE
Socket clientSocket = serverSocket.accept();

// 3. Legge il messaggio dal client
BufferedReader reader = new BufferedReader(
    new InputStreamReader(clientSocket.getInputStream()));
String message = reader.readLine();

// 4. Risponde al client
PrintWriter writer = new PrintWriter(clientSocket.getOutputStream(), true);
writer.println("ACK");

// 5. Chiude la connessione
clientSocket.close();
serverSocket.close();
```

**Punto critico**: `accept()` blocca il thread finché non arriva una connessione. In produzione e nei test va sempre eseguito su un thread separato.

---

## Lato client — ciclo di vita

```java
// 1. Si connette al server
Socket socket = new Socket("localhost", 4242);

// 2. Invia un messaggio
PrintWriter writer = new PrintWriter(socket.getOutputStream(), true);
writer.println("{\"event\":\"PULL_LOGON\",\"vaultId\":\"test-vault\"}");

// 3. Aspetta l'ACK
BufferedReader reader = new BufferedReader(
    new InputStreamReader(socket.getInputStream()));
String ack = reader.readLine();

// 4. Chiude la connessione
socket.close();
```

---

## Porta libera assegnata dall'OS

Nei test non si usa una porta fissa — si chiede all'OS una porta libera:

```java
ServerSocket serverSocket = new ServerSocket(0); // 0 = porta libera
int port = serverSocket.getLocalPort();           // porta effettivamente assegnata
```

Ogni test riceve una porta diversa — nessun conflitto tra test paralleli.

---

## Pattern corretto nei test

```java
ServerSocket serverSocket;  // attributo d'istanza — serve in più metodi

@BeforeEach
void setUp() throws IOException {
    serverSocket = new ServerSocket(0);
    int port = serverSocket.getLocalPort();

    // server su thread separato — accept() è bloccante
    new Thread(() -> {
        try {
            Socket client = serverSocket.accept();
            // leggi messaggio, rispondi ACK
            BufferedReader reader = new BufferedReader(
                new InputStreamReader(client.getInputStream()));
            reader.readLine();
            PrintWriter writer = new PrintWriter(client.getOutputStream(), true);
            writer.println("ACK");
            client.close();
        } catch (IOException e) {
            // server chiuso dal tearDown — comportamento atteso
        }
    }).start();

    // client punta alla porta assegnata dall'OS
    Properties properties = new Properties();
    properties.setProperty("socket.host", "localhost");
    properties.setProperty("socket.port", String.valueOf(port));
    socketClient = new SocketClient(properties, logService);
}

@AfterEach
void tearDown() throws IOException {
    serverSocket.close(); // interrompe accept() se ancora in attesa
}
```

---

## Perché il thread separato

```
Thread principale (test):          Thread server:
setUp()                            new Thread(() -> {
  serverSocket = new ServerSocket      serverSocket.accept() ← BLOCCA QUI
  new Thread(...).start()              // aspetta connessione
  socketClient = new SocketClient  })
test() → socketClient.send()  →→→  connessione accettata
  aspetta ACK               ←←←   risponde ACK
  termina                          client.close()
```

Senza thread separato il test si bloccherebbe su `accept()` e non raggiungerebbe mai `socketClient.send()`.

---

## Cheat-sheet

```java
new ServerSocket(0)              // porta libera scelta dall'OS
serverSocket.getLocalPort()      // porta effettivamente assegnata
serverSocket.accept()            // BLOCCANTE — aspetta connessione
new Socket("localhost", port)    // connessione client
socket.getInputStream()          // legge dal socket
socket.getOutputStream()         // scrive sul socket
new PrintWriter(out, true)       // true = autoflush — manda subito
serverSocket.close()             // chiude il server, sblocca accept()
```




---



# Jackson — Tutorial

## Cos'è

Jackson è la libreria Java standard per serializzare oggetti in JSON e deserializzare JSON in oggetti. È lo standard de facto in Spring Boot e in tutti i contesti Java enterprise.

---

## Dipendenza Maven

```xml
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.17.1</version>
</dependency>
```

---

## I tre scenari principali

### 1. Oggetto → JSON (serializzazione)

```java
ObjectMapper mapper = new ObjectMapper();

SocketMessage message = new SocketMessage("PULL_LOGON", "test-vault", 30_000L);

// oggetto → stringa JSON
String json = mapper.writeValueAsString(message);
// {"event":"PULL_LOGON","vaultId":"test-vault","timestamp":1234567890,"retryCount":0,"retryDelay":30000}

// oggetto → file
mapper.writeValue(new File("message.json"), message);

// oggetto → JSON formattato (leggibile)
String pretty = mapper.writerWithDefaultPrettyPrinter().writeValueAsString(message);
```

### 2. JSON → oggetto (deserializzazione)

```java
// stringa JSON → oggetto
String json = "{\"event\":\"PULL_LOGON\",\"vaultId\":\"test-vault\"}";
SocketMessage message = mapper.readValue(json, SocketMessage.class);

// file JSON → oggetto
SocketMessage message = mapper.readValue(new File("message.json"), SocketMessage.class);
```

### 3. JSON → struttura generica (senza model class)

```java
// quando non hai una classe target — utile per parsing flessibile
JsonNode root = mapper.readTree(json);
String event   = root.get("event").asText();
String vaultId = root.get("vaultId").asText();
long timestamp = root.get("timestamp").asLong();

// verifica se un campo esiste
if (root.has("retryCount")) {
    int count = root.get("retryCount").asInt();
}
```

---

## Requisiti sulla classe target

Per la deserializzazione Jackson richiede:

```java
// 1. costruttore senza argomenti (può essere package-private)
// 2. getter per ogni campo da serializzare
// 3. oppure annotazione @JsonProperty sui campi

public class SocketMessage {

    private final String event;

    // costruttore no-arg per Jackson
    SocketMessage() { this.event = null; }

    // costruttore di produzione
    public SocketMessage(String event, ...) { this.event = event; }

    public String getEvent() { return event; }   // serializzato come "event"
}
```

---

## Annotazioni utili

```java
// rinomina il campo nel JSON
@JsonProperty("vault_id")
private String vaultId;

// ignora il campo nella serializzazione
@JsonIgnore
private String internalState;

// gestisce campi sconosciuti nel JSON senza lanciare eccezione
@JsonIgnoreProperties(ignoreUnknown = true)
public class SocketMessage { ... }

// controlla il nome della property senza getter
@JsonProperty("event")
private String event;
```

---

## Pattern per SocketServer

Il server riceve una stringa JSON dal socket e deve:

1. Verificare che sia JSON valido
2. Estrarre il campo `event`
3. Convertirlo in `EventType`
4. Pubblicare il `SyncEvent` sulla coda

```java
private void handleMessage(String json, PrintWriter writer) {
    try {
        JsonNode root     = mapper.readTree(json);
        String eventName  = root.get("event").asText();
        String vaultId    = root.get("vaultId").asText();

        EventType type = EventType.valueOf(eventName);  // lancia IllegalArgumentException se sconosciuto
        queue.publish(new SyncEvent(type));
        writer.println(SocketResponse.ACK.name());

    } catch (IllegalArgumentException e) {
        // evento sconosciuto
        writer.println(SocketResponse.NACK.name());

    } catch (Exception e) {
        // JSON malformato o campo mancante
        writer.println(SocketResponse.ERROR.name());
    }
}
```

---

## Differenza readValue vs readTree

|Metodo|Usa quando|
|---|---|
|`readValue(json, MyClass.class)`|Hai una classe target — mapping automatico|
|`readTree(json)`|Non hai una classe target, o vuoi accesso flessibile ai campi|

Per `SocketServer` `readTree` è la scelta giusta — non hai bisogno di ricostruire un `SocketMessage`, ti serve solo estrarre `event` e `vaultId`.

---

## ObjectMapper — istanza condivisa o locale?

`ObjectMapper` è thread-safe dopo la configurazione iniziale. Creane **una sola istanza** per classe e riutilizzala:

```java
public class SocketServer {
    private static final ObjectMapper MAPPER = new ObjectMapper();
    // ...
}
```

Non creare un `new ObjectMapper()` ad ogni chiamata — è costoso da inizializzare.

---

## Cheat-sheet

```java
// serializza
mapper.writeValueAsString(obj)

// deserializza su classe
mapper.readValue(json, MyClass.class)

// deserializza su struttura generica
mapper.readTree(json)

// leggi un campo stringa
root.get("field").asText()

// leggi un campo numerico
root.get("field").asLong()
root.get("field").asInt()

// verifica esistenza
root.has("field")
root.get("field") != null && !root.get("field").isNull()
```





---





# Lambda Functions in Java — Tutorial

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
