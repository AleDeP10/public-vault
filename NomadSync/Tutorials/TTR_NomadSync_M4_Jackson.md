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

| Metodo                           | Usa quando                                                    |
| -------------------------------- | ------------------------------------------------------------- |
| `readValue(json, MyClass.class)` | Hai una classe target — mapping automatico                    |
| `readTree(json)`                 | Non hai una classe target, o vuoi accesso flessibile ai campi |

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
