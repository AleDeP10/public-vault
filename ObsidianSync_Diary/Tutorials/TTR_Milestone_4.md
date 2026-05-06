
### jqwik — panoramica

**Cos'è**: libreria Java per Property-Based Testing, si integra con JUnit 5.

**Concetto base**: invece di esempi fissi, definisci _proprietà_ che devono valere per interi domini di input. jqwik genera automaticamente centinaia di casi.

**Dipendenza Maven**:

xml

```xml
<dependency>
    <groupId>net.jqwik</groupId>
    <artifactId>jqwik</artifactId>
    <version>1.8.4</version>
    <scope>test</scope>
</dependency>
```

**Annotazioni chiave**:

|Annotazione|Significato|
|---|---|
|`@Property`|Sostituisce `@Test` — eseguito N volte con input diversi|
|`@ForAll`|Parametro generato automaticamente da jqwik|
|`@IntRange(min, max)`|Vincola il dominio a un intervallo — la tua partizione|
|`@From("nomeMetodo")`|Usa un `@Provide` custom come generatore|

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


---



# Java Local Sockets — Tutorial

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