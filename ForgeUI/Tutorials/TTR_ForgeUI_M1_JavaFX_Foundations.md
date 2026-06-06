# TTR — ForgeUI M1: JavaFX Foundations

---

## Il problema che JavaFX risolve

Un'applicazione desktop ha bisogno di disegnare pixel sullo schermo e reagire agli input dell'utente. Il sistema operativo espone API native per farlo (Win32 su Windows, Cocoa su macOS, GTK su Linux) ma sono incompatibili tra loro e impossibili da usare direttamente da Java.

JavaFX è il layer che si mette in mezzo: tu descrivi l'interfaccia in termini astratti (un bottone, un campo di testo, una finestra), JavaFX traduce quella descrizione in chiamate native per il sistema operativo corrente. Cambi OS, non cambi codice.

---

## Il modello concettuale: Scene Graph

JavaFX rappresenta l'interfaccia come un **grafo ad albero di nodi** — il Scene Graph. Ogni elemento visibile è un nodo. I nodi hanno figli. La radice dell'albero è connessa alla finestra.

```
Stage (la finestra del sistema operativo)
└── Scene (la superficie di disegno)
    └── BorderPane (nodo radice — layout)
        ├── HBox (toolbar in alto)
        │   ├── ForgeButton
        │   └── ForgeButton
        └── VBox (contenuto centrale)
            ├── ForgeTextField
            └── ForgeTextField
```

Questa struttura non è solo concettuale — è l'oggetto reale che JavaFX usa per calcolare il layout, applicare il CSS e propagare gli eventi. Se aggiungi un nodo all'albero, appare sullo schermo. Se lo rimuovi, sparisce.

---

## Stage

`Stage` è la finestra del sistema operativo. Ne esiste sempre una principale — quella che JavaFX crea e passa al metodo `start()` della tua applicazione.

```java
public class ShowcaseApp extends Application {

    @Override
    public void start(Stage primaryStage) {
        primaryStage.setTitle("ForgeUI Showcase");
        primaryStage.setWidth(800);
        primaryStage.setHeight(600);
        primaryStage.show();
    }
}
```

`show()` rende la finestra visibile. Senza `show()` la finestra esiste in memoria ma non appare. Puoi creare Stage aggiuntivi (finestre secondarie, dialog) con `new Stage()`.

---

## Scene

`Scene` è la superficie di disegno contenuta nello Stage. Ogni Scene ha esattamente un nodo radice — il contenitore top-level del tuo layout. È sulla Scene che si applicano i CSS.

```java
BorderPane root = new BorderPane();
Scene scene = new Scene(root, 800, 600);
primaryStage.setScene(scene);
```

Un'applicazione può avere più Scene ma uno Stage ne mostra una sola alla volta. Cambiare Scene significa `stage.setScene(altraScene)` — si usa raramente; più comune è cambiare il contenuto del nodo radice.

---

## I nodi

Tutto quello che vedi sullo schermo è un `Node`. La gerarchia delle classi principali:

```
Node
├── Parent                  ← può avere figli
│   ├── Region              ← ha dimensioni e background
│   │   ├── Control         ← interattivo (Button, TextField, ecc.)
│   │   └── Pane            ← layout container (HBox, VBox, BorderPane, ecc.)
│   └── Group               ← raggruppa senza layout
└── Shape                   ← forme geometriche (Rectangle, Circle, ecc.)
```

ForgeTextField e ForgeButton estendono `Control`. I layout dello showcase (`BorderPane`, `VBox`) estendono `Pane`.

---

## I layout container

JavaFX calcola automaticamente le posizioni dei figli in base al tipo di container. I principali:

**`HBox`** — dispone i figli in orizzontale, uno dopo l'altro.

**`VBox`** — dispone i figli in verticale, uno sotto l'altro.

**`BorderPane`** — divide lo spazio in cinque zone: TOP, BOTTOM, LEFT, RIGHT, CENTER. Usato tipicamente come radice della Scene — toolbar in TOP, contenuto in CENTER.

**`StackPane`** — sovrappone i figli uno sull'altro. Utile per overlay e dialog.

**`GridPane`** — griglia con righe e colonne. Per form complesse.

---

## FXML

FXML è un formato XML che descrive la struttura del Scene Graph in modo dichiarativo. Invece di costruire il layout in Java con `new VBox()`, `vbox.getChildren().add(...)` ecc., lo scrivi come markup.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<?import javafx.scene.layout.BorderPane?>
<?import javafx.scene.layout.VBox?>

<BorderPane xmlns:fx="http://javafx.com/fxml">
    <center>
        <VBox spacing="16">
            <!-- componenti qui -->
        </VBox>
    </center>
</BorderPane>
```

Il vantaggio è la separazione tra struttura (FXML) e logica (Controller Java). È l'analogo di HTML + JavaScript nel web.

### FXMLLoader

`FXMLLoader` carica il file FXML e costruisce il Scene Graph corrispondente:

```java
FXMLLoader loader = new FXMLLoader(
    getClass().getResource("/io/github/aledep10/forgeshowcase/views/main-form.fxml")
);
Parent root = loader.load();
Scene scene = new Scene(root);
```

`getResource()` cerca il file nel classpath — la barra iniziale significa "dalla root del classpath", non relativo al package della classe corrente.

### Controller

Il Controller è la classe Java collegata all'FXML. Gestisce gli eventi e manipola i nodi. Si collega con l'attributo `fx:controller` nell'FXML:

```xml
<BorderPane fx:controller="io.github.aledep10.forgeshowcase.MainController">
```

Nel controller, i nodi FXML vengono iniettati tramite `@FXML` + `fx:id`:

```xml
<!-- FXML -->
<ForgeTextField fx:id="addressField" />
```

```java
// Controller
@FXML
private ForgeTextField addressField;

@FXML
private void initialize() {
    // chiamato automaticamente da FXMLLoader dopo l'iniezione
    addressField.setLabel("Indirizzo");
}
```

---

## CSS in JavaFX

JavaFX supporta un sottoinsieme di CSS con proprietà prefissate `-fx-`. Non è CSS standard — le proprietà web come `background-color` non funzionano, si usa `-fx-background-color`.

```css
.forge-button-primary {
    -fx-background-color: #1a73e8;
    -fx-text-fill: white;
    -fx-font-size: 14px;
    -fx-padding: 8 16 8 16;
}

.forge-button-primary:hover {
    -fx-background-color: #1557b0;
}
```

Le classi CSS si applicano ai nodi con `getStyleClass().add("forge-button-primary")`.

Il CSS si applica alla Scene, non al singolo nodo — tutti i nodi figli ereditano gli stili applicabili:

```java
scene.getStylesheets().add(
    getClass().getResource("/io/github/aledep10/forgeui/themes/forge-light.css")
              .toExternalForm()
);
```

`toExternalForm()` converte la URL in una stringa nel formato che JavaFX si aspetta (`file:/...` o `jar:file:/...`).

Per swappare tema a runtime si svuota la lista e si aggiunge il nuovo CSS — esattamente quello che fa `ThemeManager.loadTheme()`.

---

## Il threading model

JavaFX ha un thread dedicato all'UI: il **JavaFX Application Thread** (anche detto FX thread). Tutte le operazioni sui nodi — lettura e scrittura di proprietà, aggiunta di figli, cambio di stile — devono avvenire su questo thread.

Se provi a modificare un nodo da un thread diverso (es. un thread di background che ha finito di caricare dati), JavaFX lancia un'eccezione.

La soluzione è `Platform.runLater()`: accoda un'operazione per essere eseguita sul FX thread non appena è disponibile.

```java
// Thread di background
new Thread(() -> {
    String data = fetchDataFromNetwork();        // operazione lenta, ok su background thread

    Platform.runLater(() -> {
        myLabel.setText(data);                  // modifica UI — deve stare qui
    });
}).start();
```

Questa regola è fondamentale per ForgeUI perché `ThemeManager` potrebbe essere chiamato da contesti diversi, e `loadTheme()` tocca `scene.getStylesheets()` — proprietà UI.

---

## Properties e Binding

JavaFX introduce il concetto di `Property` — un wrapper osservabile attorno a un valore. Quando il valore cambia, i listener vengono notificati automaticamente.

```java
StringProperty name = new SimpleStringProperty("Alessandro");

// Listener manuale
name.addListener((observable, oldValue, newValue) -> {
    System.out.println("Cambiato da " + oldValue + " a " + newValue);
});

// Binding — label si aggiorna automaticamente quando name cambia
myLabel.textProperty().bind(name);

name.set("Gabriela");   // label mostra "Gabriela" automaticamente
```

In `ForgeTextField`, il countdown dei caratteri rimanenti è aggiornato tramite un listener su `textProperty()` del TextField interno — ogni volta che l'utente digita, il listener calcola `maxLength - testo.length()` e aggiorna il label del countdown.

---

## Ciclo di vita dell'applicazione

```
Application.launch()
    → init()        ← opzionale, su thread separato, NO operazioni UI
    → start()       ← su FX thread, costruisci Stage e Scene qui
    → [app in esecuzione — event loop]
    → stop()        ← opzionale, cleanup risorse
```

`Platform.exit()` termina l'applicazione pulitamente. `System.exit()` funziona ma bypassa `stop()` — da evitare.

---

## Cheat sheet rapido

| Concetto               | Classe/metodo chiave           | Note                         |
| ---------------------- | ------------------------------ | ---------------------------- |
| Finestra               | `Stage`                        | una principale, N secondarie |
| Superficie di disegno  | `Scene`                        | uno Stage, una Scene attiva  |
| Nodo interattivo       | `Control`                      | Button, TextField, ecc.      |
| Layout verticale       | `VBox`                         | spacing configurabile        |
| Layout a zone          | `BorderPane`                   | TOP/CENTER/BOTTOM/LEFT/RIGHT |
| Carica FXML            | `FXMLLoader.load()`            | ritorna il nodo radice       |
| Applica CSS            | `scene.getStylesheets().add()` | toExternalForm() per la URL  |
| Aggiorna UI da thread  | `Platform.runLater()`          | sempre, senza eccezioni      |
| Osserva un valore      | `property.addListener()`       | old/new value nel callback   |
| Sincronizza due valori | `property.bind()`              | unidirezionale               |

