# TTR — ForgeUI M2: Design Patterns

Cheat-sheet dei pattern di progettazione usati in ForgeUI e ForgeBook, con esempi concreti dal codice e indicazioni su dove applicarli.

---

## Pattern già in uso in ForgeUI

---

### Observer / ChangeListener

**Cosa è**: un oggetto (Observer) si registra su un altro (Observable) per ricevere notifiche quando il suo stato cambia. L'Observable non sa chi lo ascolta.

**In ForgeUI**:

```java
// ForgeClearableInputSkin — ascolta il cambio di readOnly
control.readOnlyProperty().addListener((obs, oldVal, newVal) -> {
    innerField.setEditable(!newVal);
    updateClearButtonVisibility();
});
```

`obs` è l'`ObservableValue` sorgente — la property stessa. `oldVal` e `newVal` sono i valori prima e dopo. Nella maggior parte dei casi `obs` viene ignorato perché già sappiamo quale property ha scatenato l'evento. Diventa utile quando lo stesso listener è registrato su più sorgenti diverse.

**Radice JavaFX**: `ObservableValue<T>` è la base del sistema reattivo. Ogni `Property` è un `ObservableValue` — si può ascoltare, bindare, concatenare.

---

### Template Method

**Cosa è**: la classe base definisce lo scheletro di un algoritmo con step fissi. Alcuni step sono implementati dalla base, altri sono `abstract` o hook sovrascrivibili dalle sottoclassi.

**In ForgeUI**:

```java
// ForgeTextInputBaseSkin — layoutChildren è lo scheletro
protected void layoutChildren(double x, double y, double w, double h) {
    // step fissi — sempre uguali
    layoutInArea(labelNode, ...);
    layoutInArea(inputRow, ...);
    layoutInArea(helperLabel, ...);
    // hook — ogni sottoclasse aggiunge i suoi nodi extra
    layoutExtra(x, y, w, h, inputY, inputH);
}

// ForgeTextAreaSkin — sovrascrive solo il passo variabile
protected void layoutExtra(...) {
    if (getSkinnable().isResizable()) {
        layoutInArea(resizeHandle, ...);
    }
}
```

**Quando usarlo**: quando hai un algoritmo con struttura fissa ma dettagli variabili. Elimina la duplicazione mantenendo la flessibilità nei punti che cambiano.

---

### Strategy (via hook)

**Cosa è**: definisce una famiglia di algoritmi intercambiabili. Il contesto delega a un'interfaccia senza sapere quale implementazione sta usando.

**In ForgeUI**:

```java
// ForgeClearableInputSkin — isClearButtonSuppressed() è la strategy
protected boolean isClearButtonSuppressed() {
    return false;  // default: mai soppresso
}

// ForgeTextFieldSkin — strategy diversa per PASSWORD
@Override
protected boolean isClearButtonSuppressed() {
    return getSkinnable().getType() == TextFieldType.PASSWORD;
}

// ForgeComboBoxSkin (M2) — strategy diversa per dropdown aperto
@Override
protected boolean isClearButtonSuppressed() {
    return isDropdownOpen();
}
```

**Differenza da Template Method**: il Template Method cambia _parti_ di un algoritmo tramite ereditarietà. Strategy sostituisce _l'algoritmo intero_ tramite composizione o override. In JavaFX spesso si fondono — come qui.

---

### Dependency Inversion

**Cosa è**: i moduli ad alto livello non dipendono da quelli a basso livello. Entrambi dipendono da astrazioni (interfacce). Le astrazioni non dipendono dai dettagli — i dettagli dipendono dalle astrazioni.

**In ForgeUI / NomadSync**:

```java
// SyncOrchestrator non sa come notifica — dipende dall'astrazione
public interface NotificationHook {
    void onFailure(SyncEvent event, String reason);
}

// Implementazione default — scrive sul log
public class LogNotificationHook implements NotificationHook { ... }

// Implementazione futura — tray icon (M2 NomadSync)
public class TrayNotificationHook implements NotificationHook { ... }
```

**Beneficio**: l'orchestratore è testabile in isolamento con un mock. L'implementazione della tray arriva senza modificare una riga dell'orchestratore.

---

### NodeBundle (pre-super construction)

**Cosa è**: pattern specifico JavaFX per aggirare il vincolo Java che impone `super()` come prima istruzione del costruttore, quando i nodi da passare alla classe base dipendono gli uni dagli altri.

**In ForgeUI**:

```java
public class ForgeTextAreaSkin extends ForgeTextInputBaseSkin<ForgeTextArea> {

    public ForgeTextAreaSkin(ForgeTextArea control) {
        this(control, new NodeBundle());  // costruisce tutto prima del super
    }

    private ForgeTextAreaSkin(ForgeTextArea control, NodeBundle b) {
        super(control, b.clearButton, b.countdownLabel, b.labelNode,
              b.inputRow, b.helperLabel);
        // ora puoi usare this
    }

    static final class NodeBundle {
        final Label    labelNode  = new Label();
        final TextArea innerArea  = new TextArea();
        final Button   clearButton;
        final HBox     inputRow;

        NodeBundle() {
            clearButton = new Button("x");
            HBox.setHgrow(innerArea, Priority.ALWAYS);
            inputRow = new HBox(4, innerArea, clearButton);
        }
    }
}
```

---

### Dispose Chain

**Cosa è**: ogni livello di una gerarchia di oggetti rimuove le proprie risorse e delega la pulizia al livello superiore — come `finally` applicato agli oggetti.

**In ForgeUI**:

```java
// ForgeTextAreaSkin.dispose()
control.rowsProperty().removeListener(rowsListener);
labelNode.textProperty().unbind();
disposeTextBase();   // rimuove text, maxLength → chiama disposeBase()
super.dispose();     // SkinBase

// Regola: bind() → unbind()  |  addListener() → removeListener()
// Sono meccanismi distinti — non intercambiabili.
```

---

## Pattern non ancora usati — da considerare da M2 a M99+

---

### Builder + Fluent Interface

**Cosa è**: costruisce oggetti complessi passo per passo. La variante Fluent Interface restituisce `this` (o il builder) da ogni metodo, permettendo il method chaining.

```java
Accordion sidebar = Accordion.builder()
    .add("Controls")
        .add("Button")
            .story("Primary",   () -> new PrimaryButtonScene())
            .story("Secondary", () -> new SecondaryButtonScene())
        .end()
        .add("TextField")
            .story("Default",  () -> new TextFieldScene())
            .story("Password", () -> new PasswordScene())
        .end()
    .end()
    .add("Layout")
        .add("NavBar").end()
    .end()
    .build();
```

**Perché non è "stringhe concatenate"**: ogni `.add()` non aggiunge una stringa — aggiunge un nodo all'albero e restituisce un _builder figlio_ che conosce il proprio livello nella gerarchia. `.end()` risale al builder padre. È un albero di builder, non una lista piatta.

**In ForgeUI**: candidato principale per il factory di `Accordion`. Ogni `add()` restituisce un `AccordionItemBuilder`, `.end()` ritorna all'`AccordionBuilder` padre. Il livello è noto al momento della costruzione.

---

### Command

**Cosa è**: incapsula un'operazione come oggetto. Permette undo/redo, logging delle operazioni, code di operazioni.

```java
public interface AccordionCommand {
    void execute();
    void undo();
}

public class MoveItemCommand implements AccordionCommand {
    private final AccordionItem item;
    private final Accordion source, target;
    private final int position;

    public void execute() { target.addAt(item, position); source.remove(item); }
    public void undo()    { source.add(item); target.removeAt(position); }
}
```

**In ForgeBook**: indispensabile per il drag & drop con undo/redo. Ogni spostamento di accordion è un `Command` — reversibile, loggabile, serializzabile nel JSON di layout.

---

### Composite

**Cosa è**: tratta oggetti singoli e composizioni di oggetti in modo uniforme. La struttura ad albero è il caso d'uso canonico.

```java
// Accordion e AccordionItem implementano la stessa interfaccia
public interface AccordionNode {
    String getLabel();
    void render(Scene scene);
    List<AccordionNode> getChildren();
}

// Accordion — nodo composito
public class Accordion implements AccordionNode {
    private List<AccordionNode> children;  // può contenere item O altri accordion
}

// AccordionItem — foglia
public class AccordionItem implements AccordionNode {
    private List<AccordionNode> children = List.of();  // sempre vuota
}
```

**In ForgeUI**: Composite + Builder insieme costruiscono l'albero degli accordion in modo che il codice che lo attraversa non debba distinguere tra foglie e nodi.

---

### Decorator

**Cosa è**: aggiunge comportamento a un oggetto esistente senza modificarne la classe, avvolgendolo in un oggetto che implementa la stessa interfaccia.

```java
// AccordionItem base
AccordionNode item = new AccordionItem("Button");

// Decorato con metadati ForgeBook
AccordionNode annotated = new AnnotatedAccordionItem(item,
    AccordionMetadata.of(MetadataKey.STORY_REF, "controls/button/primary")
                     .and(MetadataKey.AUTHOR, "AleDeP10")
                     .and(MetadataKey.TAG, "interactive"));
```

**In ForgeUI**: la separazione `AccordionItem` / `AnnotatedAccordionItem` che abbiamo discusso — il base è un Decorator della stessa interfaccia, senza ereditarietà forzata. Il dev sceglie se usare il base o il decorato.

---

### Memento

**Cosa è**: cattura e esternalizza lo stato interno di un oggetto senza violarne l'encapsulamento, per poterlo ripristinare in seguito.

```java
public class AccordionLayoutMemento {
    private final String json;  // snapshot della struttura

    public static AccordionLayoutMemento capture(Accordion root) {
        return new AccordionLayoutMemento(serialize(root));
    }

    public void restore(Accordion root) {
        deserialize(this.json, root);
    }
}
```

**In ForgeBook**: il JSON di layout degli accordion _è_ un Memento. Ogni salvataggio è un `capture()`, ogni avvio è un `restore()`. Combinato con Command, ogni drag & drop crea un Memento — undo ripristina il Memento precedente.

---

### Registry

**Cosa è**: un registro centralizzato che tiene traccia di oggetti per nome o tipo, permettendo lookup dinamico a runtime senza dipendenze hard-coded tra i componenti.

```java
public class MetadataRegistry {

    private final Map<String, MetadataLoader> loaders = new LinkedHashMap<>();
    private final Map<String, JsonNode> cache         = new ConcurrentHashMap<>();

    // ogni consumer registra il proprio loader all'avvio
    public void register(String namespace, MetadataLoader loader) {
        loaders.put(namespace, loader);
    }

    // carica tutti i file JSON registrati
    public void loadAll() {
        loaders.forEach((ns, loader) -> cache.put(ns, loader.load()));
    }

    // restituisce i metadati per namespace
    public JsonNode get(String namespace) {
        return cache.getOrDefault(namespace, JsonNode.empty());
    }
}

// Utilizzo all'avvio di ForgeBook
registry.register("forge-ui",    () -> loadJson("forge-accordion-layout.json"));
registry.register("forge-book",  () -> loadJson("forge-book-annotations.json"));
registry.register("spotcast",    () -> loadJson("spotcast-custom-metadata.json"));
```

**File watcher integrato**: il Registry osserva i file con `WatchService` di Java NIO. Se qualcuno modifica un file JSON a runtime, il Registry invalida la cache e ricarica.

```java
// Previene il caso "qualcuno ci cambia i file sotto al culo"
WatchService watcher = FileSystems.getDefault().newWatchService();
jsonPath.getParent().register(watcher, ENTRY_MODIFY);
// al trigger → registry.reload(namespace)
```

**In ForgeBook**: ogni progetto consumer porta i suoi JSON. Il Registry li conosce tutti, li carica all'avvio, li ricarica se cambiano, e li espone per namespace. Zero accoppiamento tra i produttori di metadati.

---

### Visitor

**Cosa è**: separa un algoritmo dalla struttura su cui opera. Permette di aggiungere operazioni a una gerarchia di classi senza modificarle.

```java
public interface AccordionVisitor {
    void visit(Accordion accordion);
    void visit(AccordionItem item);
}

// Visitor che calcola la profondità massima
public class DepthVisitor implements AccordionVisitor {
    private int maxDepth = 0;
    public void visit(Accordion a) { maxDepth = Math.max(maxDepth, a.getLevel()); }
    public void visit(AccordionItem i) { /* foglia */ }
}

// Visitor che esporta in JSON
public class JsonExportVisitor implements AccordionVisitor { ... }

// Visitor che verifica l'accessibilità (ForgeBook accessibility checker)
public class A11yCheckVisitor implements AccordionVisitor { ... }
```

**In ForgeBook**: l'accessibility checker che visita ogni story è un Visitor. L'export JSON del layout è un Visitor. La validazione della struttura è un Visitor. Si aggiungono nuove operazioni senza toccare `Accordion` o `AccordionItem`.

---

## Mappa pattern → contesto ForgeUI/ForgeBook

| Pattern              | Dove si usa già                   | Dove si userà                                  |
| -------------------- | --------------------------------- | ---------------------------------------------- |
| Observer             | Skin listeners, properties JavaFX | AccordionItem state, drag events               |
| Template Method      | `layoutChildren` + `layoutExtra`  | AccordionSkin rendering pipeline               |
| Strategy             | `isClearButtonSuppressed()`       | Variant assignment, animation policy           |
| Dependency Inversion | `NotificationHook`                | AccordionPersistence, DragDropHandler          |
| NodeBundle           | Skin constructors                 | AccordionSkin                                  |
| Builder + Fluent     | —                                 | Accordion factory                              |
| Command              | —                                 | Drag & drop con undo/redo in ForgeBook         |
| Composite            | —                                 | Albero Accordion/AccordionItem                 |
| Decorator            | —                                 | AccordionItem / AnnotatedAccordionItem         |
| Memento              | —                                 | JSON layout persistence in ForgeBook           |
| Registry             | —                                 | MetadataLoader, JSON file watcher in ForgeBook |
| Visitor              | —                                 | Accessibility checker, JSON export             |