# ForgeUI — Manuale sviluppatore

---

## Cos'è ForgeUI

ForgeUI è un design system production-grade per JavaFX, distribuito come libreria open-source su Maven Central. Fornisce una gerarchia di componenti pronti all'uso, tematizzabili e accessibili, con una developer experience modellata sui moderni UI kit web (shadcn/ui, Tailwind components).

Il consumer pilota è **SpotCast** (lead tracker, nome da confermare in naming session). Ogni decisione architetturale presa durante lo sviluppo è stata validata contro requisiti di integrazione reali.

---

## Struttura del progetto

```
forge-ui/                          parent POM (packaging: pom)
├── forge-core/                    libreria pubblica — pubblicata su Maven Central
│   └── src/
│       ├── main/java/
│       │   └── io/github/aledep10/forgeui/
│       │       ├── components/    ForgeTextField, ForgeTextArea, ForgeButton, ...
│       │       │   ├── enums/     ButtonVariant, FieldState, TextFieldType, ...
│       │       │   └── skin/      ForgeTextFieldSkin, ForgeTextAreaSkin, ...
│       │       ├── config/        ForgeProperties, ForgePropertiesLoader
│       │       ├── css/           ForgePseudoClasses, ForgeStyleClasses
│       │       ├── exception/     ForgeException, ThemeException
│       │       ├── theme/         Theme, ThemeManager
│       │       └── util/          StringUtil, ValidationUtil
│       └── main/resources/
│           └── io/github/aledep10/forgeui/
│               ├── themes/        forge-light.css, forge-dark.css (M2)
│               ├── i18n/          messages_en.properties, messages_it.properties
│               └── forge.properties
├── forge-showcase/                app dimostrativa — NON pubblicata su Maven Central
└── forge-test/                    test di integrazione TestFX — NON pubblicato
```

---

## Prerequisiti

|Strumento|Versione|
|---|---|
|Java|21 LTS|
|JavaFX|21 LTS|
|Maven|3.9+|
|IntelliJ IDEA|2023.1+ (consigliato)|

---

## Build e avvio

```bash
# Build completo + test
mvn clean verify

# Solo test
mvn test -pl forge-core

# Installazione locale (rende forge-core disponibile ad altri progetti locali)
mvn install

# Avvio dello showcase
mvn javafx:run -pl forge-showcase
```

---

## Aggiungere ForgeUI come dipendenza

```xml
<dependency>
    <groupId>io.github.aledep10</groupId>
    <artifactId>forge-core</artifactId>
    <version>1.0.0</version>
</dependency>
```

> **Nota sul GroupId**: `io.github.aledep10` è un namespace provvisorio usato fino alla disponibilità del dominio ForgeUI. Una migrazione a un GroupId brandizzato (es. `io.forgeui`) è pianificata e documentata nel DTR alla voce `[ForgeUI-INFRA-002]`. Nessun contratto di stabilità è dichiarato per i rilasci SNAPSHOT.

---

## Configurazione — forge.properties

Posiziona un file `forge.properties` nella **root del classpath** dell'applicazione (`src/main/resources/forge.properties`). Tutte le chiavi sono opzionali — ForgeUI usa i default interni se il file è assente.

```properties
# Controlli — si applica a tutte le sottoclassi di ForgeInputControl
forge.control.responsive=true
forge.control.clearbutton.visibility=ON_CONTENT   # ALWAYS | ON_CONTENT | ON_FOCUS | ON_FOCUS_AND_CONTENT | NEVER

# Input testuali — si applica a ForgeTextField e ForgeTextArea
forge.textinput.countdown.visible=false

# Tema
forge.theme.default=LIGHT   # LIGHT | DARK
```

---

## Gerarchia dei componenti

```
javafx.scene.control.Control
└── ForgeInputControl          label, helperText, fieldState, required, readOnly, responsive
    └── ForgeClearableInput    contratto getText(), clearButtonVisibility
        └── ForgeTextInputBase text, maxLength, countdown, prefix/suffix
            ├── ForgeTextField type (TEXT | PASSWORD | NUMBER | EMAIL), passwordVisible
            └── ForgeTextArea  rows, wrapText, resizable

javafx.scene.control.Button
└── ForgeButton                label, accessibleLabel, variant, size, icon, loading, hideTextOnSmall
```

I controlli per cui svuotare non ha senso semantico (`ForgeSwitch`, `ForgeCheckbox` — M2+) estendono `ForgeInputControl` direttamente, bypassando `ForgeClearableInput`.

---

## Utilizzo dei componenti

### ForgeTextField

```java
ForgeTextField emailField = new ForgeTextField();
emailField.setLabel("Email");
emailField.setType(TextFieldType.EMAIL);
emailField.setMaxLength(254);
emailField.setRequired(true);
emailField.setHelperText("Inserisci un indirizzo email valido");
emailField.setCountdownVisible(true);
```

**FXML:**

```xml
<ForgeTextField label="Email" type="EMAIL" maxLength="254"
                required="true" countdownVisible="true"/>
```

**Proprietà principali:**

|Proprietà|Tipo|Default|Note|
|---|---|---|---|
|`text`|`StringProperty`|`""`|binding o set programmatico|
|`label`|`StringProperty`|`""`|visualizzata sopra il campo|
|`helperText`|`StringProperty`|`""`|suggerimento o messaggio di validazione|
|`fieldState`|`ObjectProperty<FieldState>`|`DEFAULT`|`DEFAULT \| ERROR \| WARNING \| SUCCESS`|
|`maxLength`|`IntegerProperty`|`-1`|`-1` = illimitato|
|`countdownVisible`|`BooleanProperty`|`false`|nessun effetto se `maxLength=-1`|
|`type`|`ObjectProperty<TextFieldType>`|`TEXT`|`TEXT \| PASSWORD \| NUMBER \| EMAIL`|
|`clearButtonVisibility`|`ObjectProperty<ClearButtonVisibility>`|`ON_CONTENT`|vedi enum per tutte le policy|
|`readOnly`|`BooleanProperty`|`false`|focalizzabile e selezionabile, non modificabile|
|`required`|`BooleanProperty`|`false`|solo asterisco visivo — la validazione è responsabilità del consumer|
|`responsive`|`BooleanProperty`|`true`|la label si sposta sopra il campo in layout stretto|

---

### ForgeTextArea

```java
ForgeTextArea note = new ForgeTextArea();
note.setLabel("Note");
note.setRows(5);
note.setMaxLength(1000);
note.setCountdownVisible(true);
note.setResizable(true);
```

Eredita tutte le proprietà di `ForgeTextField` tranne `type` e `passwordVisible`. Aggiunge:

|Proprietà|Tipo|Default|Note|
|---|---|---|---|
|`rows`|`IntegerProperty`|`3`|minimo `1`|
|`wrapText`|`BooleanProperty`|`true`|`false` mostra scrollbar orizzontale|
|`resizable`|`BooleanProperty`|`false`|maniglia di ridimensionamento in basso a destra|

---

### ForgeButton

```java
ForgeButton salvaButton = new ForgeButton();
salvaButton.setLabel("Salva");
salvaButton.setAccessibleLabel("Salva modifiche");   // obbligatorio — WCAG 2.1 AA
salvaButton.setVariant(ButtonVariant.PRIMARY);
salvaButton.setSize(ButtonSize.MD);
salvaButton.setOnAction(e -> handleSave());
```

**FXML:**

```xml
<ForgeButton label="Salva" accessibleLabel="Salva modifiche"
             variant="PRIMARY" size="MD"/>
```

|Proprietà|Tipo|Default|Note|
|---|---|---|---|
|`label`|`StringProperty`|`""`|testo visibile; può essere nascosto in layout stretto|
|`accessibleLabel`|`StringProperty`|`""`|**obbligatorio** — nome per gli screen reader|
|`variant`|`ObjectProperty<ButtonVariant>`|`PRIMARY`|`PRIMARY \| SECONDARY \| GHOST \| DANGER`|
|`size`|`ObjectProperty<ButtonSize>`|`MD`|`SM \| MD \| LG`|
|`icon`|`ObjectProperty<Node>`|`null`|nodo icona a sinistra del testo|
|`loading`|`BooleanProperty`|`false`|mostra spinner inline, disabilita l'interazione|
|`hideTextOnSmall`|`BooleanProperty`|`true`|nasconde il testo in layout stretto se è presente un'icona|

> **Contratto di accessibilità**: `accessibleLabel` è sempre applicato tramite `setAccessibleText()`, indipendentemente da `hideTextOnSmall` e dallo stato del layout. Un `ForgeButton` senza accessible label viola WCAG 2.1 AA.

---

## Theming

```java
ThemeManager themeManager = new ThemeManager();

// Applica un tema incluso nella libreria
themeManager.loadTheme(scene, Theme.LIGHT);

// Applica un CSS esterno (white-labelling)
themeManager.loadExternalTheme(scene, Path.of("/myapp/themes/corporate.css"));

// Persiste e ripristina la preferenza utente
themeManager.saveLastUsed(Theme.DARK);
Theme lastUsed = themeManager.getLastUsed();   // legge dalla Preferences API del SO

// Rileva il tema del sistema operativo
Theme systemTheme = themeManager.detectSystemColorScheme();   // LIGHT | DARK
```

**Temi disponibili:**

|Enum|File CSS|Stato|
|---|---|---|
|`LIGHT`|`forge-light.css`|Completo — M1|
|`DARK`|`forge-dark.css`|Pianificato — M2|
|`CUSTOM`|path esterno|API completa — M1; usa `loadExternalTheme()`|

Lo swap CSS cancella tutti i fogli di stile esistenti prima di aggiungere quello nuovo, prevenendo l'accumulo tra chiamate successive a `loadTheme()`.

---

## Architettura CSS

ForgeUI usa due meccanismi CSS complementari.

**Style classes** — applicate ai nodi figli della skin via `node.getStyleClass().add()`, identificano sotto-elementi specifici. Rappresentano _cosa è_ un nodo:

```css
.forge-field-label    { ... }
.forge-area-input     { ... }
.forge-btn-primary    { ... }
```

**Pseudo-classes** — attivate sul nodo radice del controllo via `pseudoClassStateChanged()`, rappresentano stati interattivi o di validazione. Rappresentano _come sta_ un controllo in un dato momento.

La figatona: le pseudo-classi si attivano sul nodo **radice** ma il CSS le usa per stilizzare i **nodi figli** tramite selettori discendenti. Questo significa che una sola regola CSS copre l'intera gerarchia di componenti:

```css
/* Si applica a ForgeTextField, ForgeTextArea, ForgeComboBox (M2) e tutti i futuri input control */
.forge-input-control:forge-error .forge-field-border   { -fx-border-color: #ef4444; }
.forge-input-control:forge-error .forge-field-helper   { -fx-text-fill:   #ef4444; }
```

Tutte le costanti stringa CSS sono centralizzate in `ForgePseudoClasses` e `ForgeStyleClasses` — nessuna stringa inline nel codice delle skin.

---

## Architettura skin

Ogni componente è diviso in due layer:

- **Control** (`ForgeTextField`, ecc.) — proprietà, logica di business, stato. Nessuna conoscenza del rendering.
- **Skin** (`ForgeTextFieldSkin`, ecc.) — nodi figli, layout, CSS, listener agli eventi. Nessuna logica di validazione.

La gerarchia delle skin rispecchia quella dei componenti:

```
SkinBase<C>
└── ForgeClearableInputSkin<C extends ForgeClearableInput>
    └── ForgeTextInputBaseSkin<C extends ForgeTextInputBase>
        ├── ForgeTextFieldSkin
        └── ForgeTextAreaSkin
```

**Gestione della memoria**: ogni `ChangeListener` installato nel costruttore di una skin è conservato come campo e rimosso in `dispose()`. I binding dichiarativi (`bind()`) sono rilasciati con `unbind()`. Dimenticare uno dei due impedisce al controllo di essere garbage-collected — memory leak classico JavaFX.

**Pattern NodeBundle**: i nodi passati al costruttore della classe base sono pre-costruiti in una classe statica interna `NodeBundle`, per aggirare il vincolo Java che impone `super()` come prima istruzione del costruttore.

---

## Estendere ForgeUI — aggiungere un nuovo componente

Percorso raccomandato per aggiungere un componente con clear button (es. `ForgeComboBox` in M2):

1. Crea la classe del controllo estendendo `ForgeClearableInput`.
2. Implementa `getText()` — restituisce la rappresentazione stringa del valore corrente, o `""` se vuoto.
3. Crea la skin estendendo `ForgeClearableInputSkin<TuoControllo>`.
4. Sovrascrive `isClearButtonSuppressed()` se il clear button richiede regole di soppressione aggiuntive (es. dropdown aperto).
5. Sovrascrive `onReadOnlyChanged(boolean)` per propagare lo stato read-only al nodo input interno.
6. Applica le CSS class usando le costanti di `ForgeStyleClasses`.
7. Registra nuove costanti pseudo-classe in `ForgePseudoClasses` se vengono introdotti nuovi stati.
8. Scrivi i test unitari estendendo `ForgeComponentTest`.

---

## Esecuzione dei test

```bash
mvn test -pl forge-core
```

La suite usa **JUnit 5** + **AssertJ**. I test sono puri test di property/logica — nessun scene graph JavaFX o skin coinvolti. I test di integrazione TestFX vivono in `forge-test` (non ancora popolato — M2).

Classe base: `ForgeComponentTest` inizializza il toolkit JavaFX tramite `JavaFXExtension` (inizializzazione singola con CountDownLatch).

**Copertura attuale**: 136/136 verde — `ForgeInputControl`, `ForgeClearableInput`, `ForgeTextInputBase`, `ForgeTextField`, `ForgeTextArea`, `ForgeButton`, `ThemeManager`.

---

## Roadmap

|Milestone|Scope|
|---|---|
|**M1** ✅|`ForgeTextField`, `ForgeTextArea`, `ForgeButton`, `ThemeManager`, sistema skin, `forge-light.css`|
|**M2**|`ForgeAccordion`, `ForgeSwitch`, `ForgeCheckbox`, `ForgeComboBox`, `forge-dark.css`, framework di validazione|
|**M3**|ForgeBook proof of concept (`@ForgeStory`, sidebar, widget tema/lingua, accessibility checker)|
|**M3+**|`ForgeDatePicker`, `ForgeTimePicker`, `ForgeSlider`, `ForgeColorPicker`, `ForgeRadioButton`|
|**TBD**|Pubblicazione su Maven Central, migrazione GroupId al namespace brandizzato, ForgeBook full release|

---

## Contribuire

Strategia di branching: Git Flow.

- `main` — solo release stabili taggate
- `develop` — branch di integrazione
- `feature/<nome>` — un branch per sprint

Tutte le decisioni architetturali sono documentate in `ForgeUI_DTR.md` (qui, in italiano) e `DTR_EN.md` (inglese, documentazione inclusa nel repository). Prima di introdurre una modifica strutturale, aggiungere una voce al DTR con contesto, decisione, motivazione e alternative scartate.

---

## Autori

**Alessandro De Prato** — Product Owner, Senior Software Engineer [Portfolio](https://aledep10.github.io/) [GitHub](https://github.com/AleDeP10) · [LinkedIn](https://www.linkedin.com/in/alessandro-de-prato)

**Gabriela Belmani** — Software Engineer [GitHub](https://github.com/Belmani) · [LinkedIn](https://www.linkedin.com/in/gabriela-da-sa%C3%BAde-belmani-tumfart)