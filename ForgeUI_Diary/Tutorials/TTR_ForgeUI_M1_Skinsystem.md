# TTR — ForgeUI M1: JavaFX Skin System

---

## Il problema che le Skin risolvono

Un componente JavaFX ha due responsabilità distinte che non devono mescolarsi:

**Il controllo** (`Control`) — cosa il componente _è_ e _sa fare_: le sue proprietà, la sua logica di business, il suo stato. `ForgeTextField` conosce il testo, il maxLength, il fieldState. Non sa nulla di come vengono disegnati sullo schermo.

**La skin** (`Skin`) — come il componente _appare_ e _risponde agli input_: i nodi figli, il layout, i CSS, i listener agli eventi del mouse e della tastiera. `ForgeTextFieldSkin` sa come costruire il label flottante, il countdown, il bordo colorato. Non sa nulla della logica di validazione.

Questa separazione è lo stesso principio di FXML/Controller, ma applicato a un livello più profondo — il componente custom stesso.

---

## La gerarchia delle classi

```
Skinnable (interface)
└── Control
    └── ForgeTextField          ← il tuo componente

Skin<C> (interface)
└── SkinBase<C>                 ← implementazione base con utility
    └── ForgeTextFieldSkin      ← la tua skin
```

`Control` implementa `Skinnable`, che espone `skinProperty()`. Quando chiami `createDefaultSkin()` nel controllo, JavaFX riceve la skin e la registra. Da quel momento la skin è responsabile del rendering.

---

## Il ciclo di vita

```
1. new ForgeTextField()
   → costruttore del controllo
   → properties inizializzate

2. Il controllo viene aggiunto a un Scene graph
   → JavaFX chiama createDefaultSkin()
   → ForgeTextFieldSkin viene costruita

3. ForgeTextFieldSkin(ForgeTextField control)
   → costruisce i nodi figli (TextField interno, Label countdown, ecc.)
   → installa i ChangeListener sulle properties del controllo
   → aggiunge i nodi alla skin tramite getChildren().add(...)

4. Layout pass
   → layoutChildren() viene chiamato da JavaFX ad ogni pulse
   → la skin posiziona e ridimensiona i suoi nodi figli

5. Il controllo viene rimosso dal Scene graph
   → dispose() viene chiamato sulla skin
   → la skin rimuove tutti i listener per evitare memory leak
```

Il punto 5 è critico: **ogni listener installato nella skin deve essere rimosso in `dispose()`**. Se dimentichi un listener, il controllo non viene mai garbage-collected — memory leak classico JavaFX.

---

## SkinBase — la classe da estendere

`SkinBase<C>` fornisce l'implementazione base di `Skin<C>`:

```java
public class ForgeTextFieldSkin extends SkinBase<ForgeTextField> {

    // Costruttore — riceve il controllo come parametro
    public ForgeTextFieldSkin(ForgeTextField control) {
        super(control);
        // Costruisci i nodi e installali
    }

    // Chiamato da JavaFX ad ogni layout pass
    @Override
    protected void layoutChildren(double x, double y, double w, double h) {
        // Posiziona i nodi figli nell'area disponibile
    }

    // Chiamato quando il controllo viene rimosso dal scene graph
    @Override
    public void dispose() {
        // Rimuovi tutti i listener
        super.dispose();
    }
}
```

`getChildren()` in `SkinBase` restituisce la lista dei nodi figli della skin — tutto quello che aggiungi lì viene renderizzato.

---

## Pattern: costruzione dei nodi interni

Il pattern standard per una skin con nodi interni:

```java
public class ForgeTextFieldSkin extends SkinBase<ForgeTextField> {

    // I nodi figli — campi della skin
    private final TextField innerField;
    private final Label     labelNode;
    private final Label     countdownLabel;
    private final Label     helperLabel;

    public ForgeTextFieldSkin(ForgeTextField control) {
        super(control);

        // 1. Costruisci i nodi
        innerField     = new TextField();
        labelNode      = new Label();
        countdownLabel = new Label();
        helperLabel    = new Label();

        // 2. Binding iniziale dalle properties del controllo
        labelNode.textProperty().bind(control.labelProperty());
        helperLabel.textProperty().bind(control.helperTextProperty());

        // 3. Listener per comportamenti che il binding semplice non copre
        control.textProperty().addListener((obs, old, newVal) -> updateCountdown());
        control.maxLengthProperty().addListener((obs, old, newVal) -> updateCountdown());

        // 4. Aggiungi i nodi alla skin
        getChildren().addAll(labelNode, innerField, countdownLabel, helperLabel);

        // 5. Inizializzazione stato
        updateCountdown();
    }
}
```

---

## Binding vs ChangeListener — quando usare quale

**Binding** — quando il valore del nodo figlio deve semplicemente riflettere una property del controllo, senza logica:

```java
labelNode.textProperty().bind(control.labelProperty());
// Ogni volta che control.label cambia, labelNode.text si aggiorna automaticamente
```

**ChangeListener** — quando il cambiamento richiede logica prima di aggiornare la UI:

```java
control.fieldStateProperty().addListener((obs, oldState, newState) -> {
    // Rimuovi la vecchia classe CSS, aggiungi la nuova
    helperLabel.getStyleClass().remove("forge-helper-" + oldState.name().toLowerCase());
    helperLabel.getStyleClass().add("forge-helper-" + newState.name().toLowerCase());
});
```

**Regola pratica**: preferisci il binding dove puoi. È dichiarativo, non richiede `dispose()`, e JavaFX lo ottimizza internamente. Usa il listener solo quando hai logica da eseguire.

---

## layoutChildren — il metodo di layout

`layoutChildren(x, y, w, h)` riceve l'area disponibile per il contenuto (già sottratti i padding/insets) e deve posizionare i nodi figli all'interno:

```java
@Override
protected void layoutChildren(double x, double y, double w, double h) {
    // Label sopra il campo
    double labelHeight = labelNode.prefHeight(w);
    layoutInArea(labelNode, x, y, w, labelHeight,
                 0, HPos.LEFT, VPos.TOP);

    // Campo di testo sotto la label
    double fieldY = y + labelHeight + 4; // 4px gap
    double fieldH = innerField.prefHeight(w);
    layoutInArea(innerField, x, fieldY, w, fieldH,
                 0, HPos.LEFT, VPos.TOP);

    // Countdown in alto a destra del bordo del campo
    double countdownW = countdownLabel.prefWidth(-1);
    layoutInArea(countdownLabel, x + w - countdownW, fieldY,
                 countdownW, fieldH, 0, HPos.RIGHT, VPos.CENTER);
}
```

`layoutInArea` è il metodo di `SkinBase` che posiziona un nodo nell'area specificata rispettando alignment e baseline.

---

## dispose — rimozione dei listener

Ogni `ChangeListener` aggiunto in costruzione va rimosso in `dispose()`. Il pattern standard è conservare un riferimento al listener:

```java
// Campo della skin
private final ChangeListener<String> textListener;

// Nel costruttore
textListener = (obs, old, newVal) -> updateCountdown();
control.textProperty().addListener(textListener);

// In dispose()
@Override
public void dispose() {
    getSkinnable().textProperty().removeListener(textListener);
    super.dispose();
}
```

Alternativa moderna con `InvalidationListener` e lambda — meno verbosa ma richiede lo stesso pattern di conservazione del riferimento.

---

## CSS nelle Skin — pseudo-classi e gerarchia

Per applicare stili diversi in base allo stato del controllo (es. bordo rosso in ERROR), JavaFX usa le **pseudo-classi CSS**.

### Dove definire le pseudo-classi

Le pseudo-classi legate a `fieldState` appartengono a `ForgeInputControl` — non alle skin dei singoli componenti. `fieldState` è una property della classe base, quindi la sua rappresentazione CSS deve vivere lì. Ogni skin concreta attiva o disattiva le pseudo-classi ereditate senza ridichiarare nulla.

```java
// In ForgeInputControl — static final condivise da tutta la gerarchia
public static final PseudoClass PSEUDO_CLASS_ERROR   =
        PseudoClass.getPseudoClass("forge-error");
public static final PseudoClass PSEUDO_CLASS_WARNING =
        PseudoClass.getPseudoClass("forge-warning");
public static final PseudoClass PSEUDO_CLASS_SUCCESS =
        PseudoClass.getPseudoClass("forge-success");
```

Il listener che attiva le pseudo-classi va installato in `ForgeInputControl` stesso — una volta sola per tutta la gerarchia:

```java
// Nel costruttore di ForgeInputControl
fieldState.addListener((obs, oldState, newState) -> {
    pseudoClassStateChanged(PSEUDO_CLASS_ERROR,   newState == FieldState.ERROR);
    pseudoClassStateChanged(PSEUDO_CLASS_WARNING, newState == FieldState.WARNING);
    pseudoClassStateChanged(PSEUDO_CLASS_SUCCESS, newState == FieldState.SUCCESS);
});
```

### Selettori CSS — classe base, non classe concreta

Il selettore CSS usa la classe base `.forge-input-control`, non i nomi dei componenti specifici. Ogni componente ForgeUI applica questa classe nel proprio costruttore:

```java
// In ForgeInputControl — applicata una volta sola per tutta la gerarchia
getStyleClass().add("forge-input-control");
```

Le regole CSS scritte sulla classe base si propagano automaticamente a tutti i componenti figli — `ForgeTextField`, `ForgeTextArea`, `ForgeComboBox` (M2), `ForgeDatePicker` (M3+) — senza duplicare nulla:

```css
/* Regola valida per TUTTI i ForgeInputControl in stato ERROR */
.forge-input-control:forge-error .forge-field-border {
    -fx-border-color: #e53e3e;
}

.forge-input-control:forge-warning .forge-field-border {
    -fx-border-color: #dd6b20;
}

.forge-input-control:forge-success .forge-field-border {
    -fx-border-color: #38a169;
}

/* Helper text — stesso meccanismo */
.forge-input-control:forge-error .forge-helper-text {
    -fx-text-fill: #e53e3e;
}
```

### Perché non scrivere `.forge-text-field:forge-error`

Se legassi la pseudo-classe al nome del componente specifico, dovresti riscrivere la stessa regola per ogni componente futuro:

```css
/* ❌ Da evitare — duplicazione garantita a ogni nuovo componente */
.forge-text-field:forge-error  .forge-field-border { -fx-border-color: #e53e3e; }
.forge-text-area:forge-error   .forge-field-border { -fx-border-color: #e53e3e; }
.forge-combo-box:forge-error   .forge-field-border { -fx-border-color: #e53e3e; }

/* ✅ Corretto — scritto una volta, valido per tutta la gerarchia */
.forge-input-control:forge-error .forge-field-border { -fx-border-color: #e53e3e; }
```

È lo stesso principio dell'ereditarietà Java applicato al CSS — la regola vive al livello più alto della gerarchia dove il concetto ha senso.

---

## Cheat sheet rapido

| Concetto            | Classe/metodo                      | Note                                       |
| ------------------- | ---------------------------------- | ------------------------------------------ |
| Classe base skin    | `SkinBase<C>`                      | estendi questa, non `Skin<C>` direttamente |
| Aggiungi nodi       | `getChildren().add(node)`          | nel costruttore della skin                 |
| Binding semplice    | `node.prop().bind(control.prop())` | preferisci al listener                     |
| Listener con logica | `control.prop().addListener(...)`  | conserva riferimento per dispose           |
| Rimuovi listener    | `dispose()` + `removeListener()`   | obbligatorio — evita memory leak           |
| Posiziona nodi      | `layoutInArea(...)`                | in `layoutChildren()`                      |
| Stile condizionale  | `pseudoClassStateChanged()`        | collega stato Java a CSS                   |
| Accedi al controllo | `getSkinnable()`                   | dentro la skin                             |
