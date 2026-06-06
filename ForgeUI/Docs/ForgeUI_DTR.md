# ForgeUI — Decision Track Record

> Documento consolidato. Aggiornato al termine di ogni milestone. Formato per decisione: **Contesto → Decisione → Motivazione → Alternative scartate → Stato**

---

## Indice

- [Infrastruttura e progetto](https://claude.ai/chat/cddb3488-97d2-4791-8cd6-aba3aa40997d#infrastruttura-e-progetto)
- [Architettura componenti](https://claude.ai/chat/cddb3488-97d2-4791-8cd6-aba3aa40997d#architettura-componenti)
- [Sistema skin](https://claude.ai/chat/cddb3488-97d2-4791-8cd6-aba3aa40997d#sistema-skin)
- [CSS e pseudo-classi](https://claude.ai/chat/cddb3488-97d2-4791-8cd6-aba3aa40997d#css-e-pseudo-classi)
- [Configurazione e properties](https://claude.ai/chat/cddb3488-97d2-4791-8cd6-aba3aa40997d#configurazione-e-properties)
- [Theming](https://claude.ai/chat/cddb3488-97d2-4791-8cd6-aba3aa40997d#theming)
- [Eccezioni](https://claude.ai/chat/cddb3488-97d2-4791-8cd6-aba3aa40997d#eccezioni)
- [Validazione M2+](https://claude.ai/chat/cddb3488-97d2-4791-8cd6-aba3aa40997d#validazione-m2)
- [ForgeBook M3+](https://claude.ai/chat/cddb3488-97d2-4791-8cd6-aba3aa40997d#forgebook-m3)

---

## Infrastruttura e progetto

---

### [ForgeUI-INFRA-001] Struttura Maven multi-module dal primo commit

**Contesto**: il progetto è destinato a tre moduli distinti — `forge-core` (libreria pubblica), `forge-showcase` (app dimostrativa), `forge-test` (test automatizzati). La struttura Maven non è refactorable in modo indolore dopo i primi commit.

**Decisione**: multi-module Maven con parent POM dal primo commit.

**Motivazione**: `forge-showcase` dipende da `forge-core` come consumer esterno fin dal giorno 1, validando per design l'esperienza di integrazione della libreria senza richiedere un progetto separato fittizio. Costo iniziale minimo (un POM aggiuntivo).

**Alternativa scartata**: single-module iniziale con refactor successivo — rimanda il problema, rischia dipendenze circolari difficili da districare.

---

### [ForgeUI-INFRA-002] GroupId provvisorio `io.github.aledep10`

**Contesto**: Maven Central richiede un namespace verificabile. Il brand definitivo di ForgeUI non ha ancora un dominio dedicato. Le finanze attuali non giustificano la spesa per dominio + hosting.

**Decisione**: `io.github.aledep10` per i primi rilasci SNAPSHOT. Migrazione a namespace brandizzato (es. `io.forgeui`) appena il dominio sarà disponibile.

**Motivazione**: verificabile gratuitamente tramite GitHub senza possedere un dominio. Non blocca la pubblicazione su Maven Central. La migrazione è un'operazione documentata e gestibile.

**Rischi accettati**: chi integra ForgeUI in SNAPSHOT con il groupId provvisorio dovrà aggiornare la dipendenza alla migrazione. Mitigato dalla natura SNAPSHOT — nessun contratto di stabilità dichiarato verso i consumer.

---

### [ForgeUI-INFRA-003] JavaFX 21 LTS come target

**Contesto**: tutti i progetti del portfolio usano Java 21 LTS. JavaFX ha release cadence separata da OpenJDK (separato dal 2018).

**Decisione**: JavaFX 21 LTS.

**Motivazione**: allineamento con Java 21 LTS; stabilità garantita per il ciclo di vita previsto; ampia adozione enterprise.

**Alternativa scartata**: JavaFX 23/24 — nessun LTS, rischio breaking changes nei moduli interni.

**Nota**: `Platform.getPreferences().getColorScheme()` è disponibile solo da JavaFX 22+, non da JavaFX 21. Il rilevamento del tema di sistema è delegato a `jSystemThemeDetector` come libreria di supporto cross-platform.

---

### [ForgeUI-INFRA-004] FXML come standard di layout in forge-showcase

**Contesto**: i layout UI possono essere definiti programmaticamente in Java o dichiarativamente in FXML.

**Decisione**: FXML per i layout di `forge-showcase`.

**Motivazione**: separazione netta layout/logica (analogo a JSX/TSX nel portfolio web); testabilità indipendente del controller; leggibilità immediata della struttura gerarchica; compatibilità con Scene Builder per iterazioni visive rapide.

**Alternativa scartata**: layout programmatico puro — verboso, mescola struttura e logica, difficile da revisionare visivamente.

---

### [ForgeUI-INFRA-005] Git Flow come strategia di branching

**Contesto**: il progetto ha milestone sequenziali con rilasci stabili su `main` e sviluppo continuo su `develop`.

**Decisione**: Git Flow standard — `main` per release taggate, `develop` per integrazione continua, `feature/<nome>` per ogni sprint.

**Motivazione**: separa chiaramente il codice stabile dal codice in sviluppo; ogni milestone corrisponde a una release taggata su `main`.

---

## Architettura componenti

---

### [ForgeUI-ARCH-001] Gerarchia di componenti a quattro livelli

**Contesto**: i componenti di input condividono molte proprietà (label, helperText, fieldState, readOnly, responsive). È necessario evitare la duplicazione senza creare un unico super-componente monolitico.

**Decisione**:

```
javafx.scene.control.Control
└── ForgeInputControl          (label, helperText, fieldState, required, readOnly, responsive)
    └── ForgeClearableInput    (getText() abstract, clearButtonVisibility)
        └── ForgeTextInputBase (text, maxLength, countdown, prefix/suffix)
            ├── ForgeTextField (type, passwordVisible)
            └── ForgeTextArea  (rows, wrapText, resizable)
```

`ForgeButton` estende `javafx.scene.control.Button` direttamente — non è un input control.

**Motivazione**: ogni livello aggiunge esattamente le proprietà che hanno senso a quel livello di astrazione. Pattern identico a `TextInputControl → TextField / TextArea` di JavaFX stesso.

**Alternative scartate**:

- `ForgeTextField` con `type=TEXTAREA`: `ForgeTextField` con type textarea mentirebbe sul suo nome; `TextField` e `TextArea` di JavaFX hanno API di layout incompatibili.
- Interfaccia `Clearable` + composizione: in JavaFX le Properties devono essere dichiarate sul nodo per funzionare con FXML e il binding — un oggetto innestato rompe entrambi.

---

### [ForgeUI-ARCH-002] `getText()` astratto in `ForgeClearableInput`

**Contesto**: `ForgeClearableInputSkin.updateClearButtonVisibility()` deve sapere se c'è contenuto da cancellare, senza conoscere l'implementazione interna di ogni sottoclasse.

**Decisione**: `getText()` dichiarato `abstract` in `ForgeClearableInput`. Ogni sottoclasse lo implementa in base al proprio modello dati.

**Motivazione**: il clear button ha senso semantico solo quando c'è contenuto da cancellare. `getText()` è la precondizione del clear button, non una feature testuale.

**Implementazioni previste**:

- `ForgeTextInputBase`: `return text.get()`
- `ForgeComboBox` (M2): `return selectedItem != null ? selectedItem.toString() : ""`
- `ForgeDatePicker` (M3+): `return` stringa data formattata, o `""` se nessuna data selezionata

**Regola**: il valore restituito non deve mai essere `null` — restituire `""` per rappresentare lo stato vuoto/non impostato.

---

### [ForgeUI-ARCH-003] `ForgeTextField` come `Control` composito, non estensione di `TextField`

**Contesto**: label flottante, countdown, helper text, prefix/suffix richiedono nodi figli che non sono gestibili estendendo direttamente `TextField`.

**Decisione**: `ForgeTextField` estende `Control`; usa `TextField` internamente come nodo delegato con binding su `textProperty()`.

**Motivazione**: pieno controllo sul layout; possibilità di aggiungere nodi figli senza hackerare la gerarchia interna di JavaFX; pattern usato da librerie mature come ControlsFX e AtlantaFX.

**Alternativa scartata**: estensione diretta di `TextField` — non permette nodi figli nel layout senza override non documentati; CSS più difficile da isolare.

---

### [ForgeUI-ARCH-004] `label` e `helperText` in `ForgeInputControl`, non in `ForgeTextInputBase`

**Contesto**: la decisione iniziale era tenere `label` e `helperText` in `ForgeTextInputBase`. Con l'arrivo di `ForgeSwitch` e `ForgeCheckbox` (M2), anche questi componenti avranno label e helper.

**Decisione**: `label` e `helperText` salgono in `ForgeInputControl`.

**Motivazione**: qualsiasi `ForgeInputControl` ha una label e un testo helper — non ha senso limitarli ai controlli testuali. Il refactoring è avvenuto durante M1 con l'introduzione di `ForgeInputControl` come layer dedicato.

---

### [ForgeUI-ARCH-005] `ForgeButton` estende `Button`, non `ForgeInputControl`

**Contesto**: `ForgeButton` è un trigger di azione, non un input control. Non ha `fieldState`, `readOnly`, `helperText`.

**Decisione**: `ForgeButton` estende `javafx.scene.control.Button` direttamente.

**Motivazione**: separazione semantica netta — i pulsanti non raccolgono input, li attivano. Estendere `ForgeInputControl` avrebbe inquinato l'API pubblica con proprietà prive di significato.

---

### [ForgeUI-ARCH-006] `accessibleLabel` obbligatorio su `ForgeButton`

**Contesto**: `hideTextOnSmall=true` nasconde il testo visivo in layout compatto. Un pulsante senza testo visivo è inaccessibile agli screen reader senza alternativa testuale.

**Decisione**: `accessibleLabel` sempre valorizzato tramite `setAccessibleText()`, indipendente da `hideTextOnSmall`. Il developer consumer può disattivare `hideTextOnSmall` per istanza, ma non può omettere `accessibleLabel`.

**Motivazione**: WCAG 2.1 AA richiede che ogni controllo interattivo abbia un nome accessibile. Pattern analogo ad `aria-label` nel web kit del portfolio TodoList.

---

### [ForgeUI-ARCH-007] `type=PASSWORD` — swap del nodo interno invece di componente separato

**Contesto**: JavaFX ha `PasswordField` come classe separata, non come proprietà di `TextField`. È necessario decidere come gestire il tipo PASSWORD in `ForgeTextField`.

**Decisione**: `ForgeTextField` con `type=PASSWORD` sostituisce a runtime il nodo interno `TextField` con un `PasswordField`. Il `textListener` viene migrato al nuovo nodo al momento dello swap.

**Motivazione**: API uniforme per il consumer — usa sempre `ForgeTextField` indipendentemente dal tipo; la skin gestisce la complessità dello swap in modo trasparente.

**Alternativa scartata**: `ForgePasswordField` separato — duplicazione di tutte le proprietà comuni; il consumer deve ricordare quale classe usare per quale tipo.

---

### [ForgeUI-ARCH-008] `passwordVisibleProperty()` esposto come pubblico

**Contesto**: la skin è in un package separato (`skin`) rispetto al controllo (`components`). La property `passwordVisible` deve essere accessibile dalla skin.

**Decisione**: `passwordVisibleProperty()` esposto come `public` — coerente con tutte le altre property del controllo. Le mutazioni passano obbligatoriamente per `togglePasswordVisibility()`.

**Motivazione**: la visibilità package-private avrebbe richiesto di spostare la skin nello stesso package del controllo — scelta che mescola responsabilità. La coerenza con il resto dell'API è più importante dell'incapsulamento forzato.

**Nota**: `togglePasswordVisibility()` è il punto di accesso semanticamente corretto per le mutazioni. La property pubblica è per l'osservazione.

---

## Sistema skin

---

### [ForgeUI-SKIN-001] Gerarchia skin speculare alla gerarchia componenti

**Contesto**: `ForgeClearableInputSkin.updateClearButtonVisibility()` e `ForgeTextInputBaseSkin.updateCountdown()` sarebbero duplicati in `ForgeTextFieldSkin` e `ForgeTextAreaSkin` se ogni skin estendesse direttamente `SkinBase`.

**Decisione**:

```
SkinBase<C>
└── ForgeClearableInputSkin<C extends ForgeClearableInput>
    (clearButton, readOnly, focus, updateClearButtonVisibility(), isClearButtonSuppressed(), onReadOnlyChanged())
    └── ForgeTextInputBaseSkin<C extends ForgeTextInputBase>
        (countdownLabel, text, maxLength, updateCountdown(), layoutChildren, computePrefHeight)
        ├── ForgeTextFieldSkin
        └── ForgeTextAreaSkin
```

`ForgeButtonSkin` estende `SkinBase<ForgeButton>` direttamente.

**Motivazione**: zero duplicazione di `updateClearButtonVisibility()` e `updateCountdown()`; la gerarchia skin rispecchia quella dei controlli rendendo il codice prevedibile; aggiungere `ForgeComboBoxSkin` in M2 richiederà solo di estendere `ForgeClearableInputSkin` e sovrascrivere `isClearButtonSuppressed()`.

---

### [ForgeUI-SKIN-002] Hook `isClearButtonSuppressed()` per soppressione component-specific

**Contesto**: il clear button deve essere soppresso in contesti specifici per componente che vanno oltre i casi generici (readOnly, disabled): PASSWORD in `ForgeTextField`, dropdown aperto in `ForgeComboBox`, calendario aperto in `ForgeDatePicker`.

**Decisione**: hook `protected boolean isClearButtonSuppressed()` in `ForgeClearableInputSkin` con implementazione di default che restituisce `false`. Ogni skin concreta sovrascrive il metodo per aggiungere le proprie regole.

**Override previsti**:

|Skin|Condizione di soppressione|
|---|---|
|`ForgeTextFieldSkin`|`type == PASSWORD`|
|`ForgeComboBoxSkin` (M2)|dropdown aperto|
|`ForgeDatePickerSkin` (M3+)|calendario aperto|

---

### [ForgeUI-SKIN-003] Hook `onReadOnlyChanged(boolean)` per propagazione read-only

**Contesto**: il cambio di `readOnly` deve essere propagato al nodo input interno di ogni skin (`TextField`, `TextArea`, `ComboBox`...), ma `ForgeClearableInputSkin` non conosce il nodo interno.

**Decisione**: hook `protected void onReadOnlyChanged(boolean readOnly)` in `ForgeClearableInputSkin` con implementazione no-op di default. Ogni skin concreta sovrascrive per propagare al proprio nodo interno.

---

### [ForgeUI-SKIN-004] `layoutChildren` e `computePrefHeight` centralizzati in `ForgeTextInputBaseSkin`

**Contesto**: il layout a tre righe (label, input, helper) è identico in `ForgeTextFieldSkin` e `ForgeTextAreaSkin`. L'unica differenza è il resize handle di `ForgeTextAreaSkin`.

**Decisione**: `layoutChildren` e `computePrefHeight` implementati in `ForgeTextInputBaseSkin` con due hook aggiuntivi:

- `layoutExtra(x, y, w, h, inputY, inputH)` — no-op di default, sovrascritta da `ForgeTextAreaSkin` per il resize handle
- `extraPrefHeight(width)` — ritorna `0` di default, sovrascritta se necessario

**Motivazione**: eliminazione della duplicazione rilevata da IntelliJ IDEA ("Duplicated code fragment, 19 lines long").

---

### [ForgeUI-SKIN-005] Pattern `NodeBundle` per la costruzione pre-`super()`

**Contesto**: Java impone che `super()` sia la prima istruzione in un costruttore. I nodi passati alla classe base devono essere costruiti prima della chiamata, ma alcuni dipendono da altri (es. `inputRow` dipende da `innerArea` e `clearButton`).

**Decisione**: classe statica interna `NodeBundle` che costruisce tutti i nodi interdipendenti prima della chiamata a `super()`. Il costruttore privato delegante riceve il bundle e lo smista.

```java
public ForgeTextAreaSkin(ForgeTextArea control) {
    this(control, new NodeBundle());
}
private ForgeTextAreaSkin(ForgeTextArea control, NodeBundle b) {
    super(control, b.clearButton, b.countdownLabel, b.labelNode, b.inputRow, b.helperLabel);
    // ...
}
```

**Nota**: `NodeBundle` è package-private (non `private`) per permettere referenze nei Javadoc.

---

### [ForgeUI-SKIN-006] `dispose()` a catena — `disposeTextBase()` → `disposeBase()` → `super.dispose()`

**Contesto**: ogni livello della gerarchia skin installa listener che devono essere rimossi per evitare memory leak quando il controllo viene rimosso dal scene graph.

**Decisione**: catena di dispose esplicita. Ogni classe concreta rimuove i propri listener e poi chiama il metodo del livello sopra:

```java
// ForgeTextAreaSkin.dispose()
control.rowsProperty().removeListener(rowsListener);
// ...
labelNode.textProperty().unbind();
disposeTextBase();  // rimuove text, maxLength → poi chiama disposeBase()
super.dispose();
```

**Regola**: `bind()` dichiarativi si rilasciano con `unbind()` (non `removeListener()`). `addListener()` si rilascia con `removeListener()`. Sono meccanismi distinti.

---

## CSS e pseudo-classi

---

### [ForgeUI-CSS-001] Pseudo-classi di validazione in `ForgeInputControl`, non nelle skin

**Contesto**: le pseudo-classi `:forge-error`, `:forge-warning`, `:forge-success` devono applicarsi a qualsiasi `ForgeInputControl` — TextField, TextArea, ComboBox, Checkbox, ecc.

**Decisione**: le tre `PseudoClass` pubbliche (`PSEUDO_CLASS_ERROR`, `PSEUDO_CLASS_WARNING`, `PSEUDO_CLASS_SUCCESS`) dichiarate in `ForgeInputControl`. Il listener su `fieldState` che le attiva installato nel costruttore di `ForgeInputControl` — una volta sola per tutta la gerarchia.

**CSS target**: `.forge-input-control:forge-error { ... }` — una regola copre tutta la gerarchia.

**Alternativa scartata**: pseudo-classi dichiarate nelle singole skin — richiederebbe di replicare la regola CSS per ogni componente futuro.

---

### [ForgeUI-CSS-002] `ForgePseudoClasses` — registro centralizzato

**Contesto**: le pseudo-classi CSS erano dichiarate come `static final` locali nelle skin, con rischio di duplicazione e typo nelle stringhe.

**Decisione**: classe `ForgePseudoClasses` nel package `io.github.aledep10.forgeui.css` con costanti raggruppate per area funzionale: `State` (error, warning, success), `Interaction` (focused, hovered, pressed, loading), `Layout` (compact), `Input` (readonly, password-visible).

**Motivazione**: ogni stringa letterale CSS dichiarata esattamente una volta; refactoring sicuro; typo impossibili a runtime.

---

### [ForgeUI-CSS-003] `ForgeStyleClasses` — registro centralizzato CSS style classes

**Contesto**: le stringhe CSS per le style classes dei nodi figli (`.forge-field-label`, `.forge-area-input`, ecc.) erano hardcodate nelle skin.

**Decisione**: classe `ForgeStyleClasses` nel package `io.github.aledep10.forgeui.css` con costanti raggruppate per componente: `Shared`, `TextField`, `TextArea`, `Button`.

**Distinzione da `ForgePseudoClasses`**:

- **Style classes**: applicate ai nodi figli della skin via `getStyleClass().add()`. Target: `.forge-field-label`.
- **Pseudo-classes**: attivate sul nodo radice del controllo via `pseudoClassStateChanged()`. Target: `:forge-error`.

---

### [ForgeUI-CSS-004] `forge-countdown-exhausted` è una style class, non una pseudo-classe

**Contesto**: discussione su dove collocare la costante `forge-countdown-exhausted`.

**Decisione**: `ForgeStyleClasses.Shared.COUNTDOWN_EXHAUSTED` — è una CSS style class gestita via `getStyleClass().add/remove()` sul nodo `countdownLabel`, non una pseudo-classe attivata sul controllo radice.

**Regola**: le pseudo-classi rappresentano stati del controllo radice; le style classes rappresentano stati dei nodi figli della skin.

---

### [ForgeUI-CSS-005] `forge-focused` invece di `:focused` nativo JavaFX

**Contesto**: la pseudo-classe `:focused` nativa di JavaFX si applica solo al nodo radice del controllo. I nodi figli della skin (es. il bordo) non possono essere stilizzati da `:focused` del genitore senza workaround.

**Decisione**: pseudo-classe custom `forge-focused` attivata dalla skin tramite `pseudoClassStateChanged()` al cambio di focus del controllo.

**Motivazione**: permette di stilizzare qualsiasi nodo figlio della skin in risposta al focus del controllo, con selettori CSS del tipo `.forge-input-control:forge-focused .forge-field-border { ... }`.

---

## Configurazione e properties

---

### [ForgeUI-CFG-001] `ForgePropertiesLoader` — loader centralizzato con fallback silenzioso

**Contesto**: ogni classe della gerarchia (`ForgeInputControl`, `ForgeClearableInput`, `ForgeTextInputBase`) legge valori da `forge.properties`. Senza un loader centralizzato ogni classe implementa la propria logica di caricamento.

**Decisione**: `ForgePropertiesLoader` come utility class con metodi statici `get()`, `getBoolean()`, `getEnum()`. Restituisce sempre un valore di default se la chiave è assente o il file non esiste. Non lancia mai eccezioni — fallback silenzioso con log `WARNING`.

**Motivazione**: eliminazione dei try-catch duplicati; comportamento prevedibile anche quando `forge.properties` è assente dal classpath.

---

### [ForgeUI-CFG-002] `ForgeProperties` — registro centralizzato delle chiavi

**Contesto**: le chiavi di `forge.properties` (`forge.control.responsive`, `forge.control.clearbutton.visibility`, ecc.) erano stringhe hardcodate nei costruttori.

**Decisione**: classe `ForgeProperties` con costanti raggruppate in nested classes: `Control`, `TextInput`, `Theme`.

**Motivazione**: una sola fonte di verità per tutte le chiavi; typo impossibili; refactoring sicuro.

---

### [ForgeUI-CFG-003] `forge.theme.last_used` in `Preferences` API, non in `forge.properties`

**Contesto**: il tema selezionato dall'utente deve essere persistito tra le sessioni. Due costanti sembravano simili: `DEFAULT_THEME` (in `forge.properties`) e la chiave Preferences.

**Decisione**: separazione netta tra i due layer:

- `forge.theme.default` in `forge.properties` → tema al primo avvio, senza preferenza utente persistita
- `forge.theme.last_used` come chiave nella `Preferences` API (Windows registry / `~/.java/.userPrefs`) → preferenza utente a runtime

**Motivazione**: sono due concetti distinti. `DEFAULT_THEME` è una decisione del developer. La chiave Preferences è una decisione dell'utente. Mescolarli creerebbe ambiguità nella logica di selezione del tema iniziale.

---

## Theming

---

### [ForgeUI-THEME-001] ThemeManager con CSS swap su Scene + Preferences API

**Contesto**: necessario un meccanismo per switchare tema a runtime, rilevare il tema di sistema al primo avvio, e persistere la preferenza utente.

**Decisione**: `ThemeManager` con `scene.getStylesheets().clear()` + `add()` per lo swap; `java.util.prefs.Preferences` per la persistenza; `jSystemThemeDetector` per il rilevamento del tema di sistema (cross-platform: Windows 10+, macOS Mojave+, Gnome, KDE Plasma); fallback a LIGHT se il rilevamento fallisce.

**Motivazione**: pattern standard JavaFX; zero dipendenze esterne per il core del theming; `Preferences` è cross-platform; il fallback garantisce comportamento prevedibile su qualsiasi JVM incluse headless.

---

### [ForgeUI-THEME-002] Fallback di `detectSystemColorScheme()` hardcoded a LIGHT

**Contesto**: il fallback di `detectSystemColorScheme()` potrebbe sembrare candidato a leggere `forge.theme.default` invece di hardcodare `LIGHT`.

**Decisione**: fallback hardcoded a `LIGHT` in `detectSystemColorScheme()`. `forge.theme.default` entra in gioco nella logica di selezione del tema iniziale a un livello superiore.

**Motivazione**: `detectSystemColorScheme()` ha un compito specifico: rilevare il tema del sistema operativo. Il suo fallback è una rete di sicurezza per ambienti headless o OS non supportati, non una preferenza configurabile. Usare `DEFAULT_THEME` come fallback cambierebbe semantica in modo inatteso.

---

### [ForgeUI-THEME-003] API temi custom `loadExternalTheme(scene, path)`

**Contesto**: i developer consumer potrebbero voler applicare CSS personalizzati senza modificare `forge-core`.

**Decisione**: `ThemeManager.loadExternalTheme(Scene scene, Path cssPath)` — carica un CSS da path esterno, sostituendo i fogli di stile correnti della scene.

**Motivazione**: ispirato al sistema di template Joomla/Oxygen del portfolio australiano; permette white-labeling completo senza fork della libreria.

**Stato M1**: implementato come API completa (non solo scaffolding).

---

## Eccezioni

---

### [ForgeUI-EXC-001] Gerarchia eccezioni: `ForgeException → ThemeException`

**Contesto**: le operazioni di theming (CSS non trovato, path non valido) devono lanciare eccezioni con messaggi significativi.

**Decisione**: `ForgeException extends RuntimeException` come base; `ThemeException extends ForgeException` per errori specifici del theming.

**Motivazione**: unchecked exceptions — il consumer non è obbligato a gestirle esplicitamente ma può farlo; messaggi di errore contestuali invece di `NullPointerException` generici.

**Roadmap eccezioni future**:

- `ForgeValidationException` (M2) — errori del framework di validazione
- `ForgeConfigException` — se `ForgePropertiesLoader` dovrà segnalare configurazioni non valide

---

## Validazione M2+

---

### [ForgeUI-VAL-001] Architettura validazione: ibrido in-component + ForgeForm

**Contesto**: in React lo stato di validazione vive in un hook centralizzato perché React non ha binding reattivo nativo. In JavaFX il binding reattivo nativo permette un'architettura più componibile.

**Decisione** (pianificata per M2): architettura ibrida.

- **Opzione A** — ogni `ForgeTextInputBase` gestisce la propria validazione; il developer registra validators sul campo direttamente
- **Opzione B** — `ForgeForm` aggrega `isValid()` tramite binding su tutti i campi figli
- **Scelta**: **ibrido A+B** — i campi sono autonomi (A), `ForgeForm` opzionale aggrega per i binding sui pulsanti (B)

```java
// A — validators sul campo
nameField.addValidator(Validators.required());
emailField.addValidator(Validators.required(), Validators.email());

// B — binding automatico tramite ForgeForm
saveButton.disableProperty().bind(form.validProperty().not());
```

**Motivazione**: JavaFX ha il binding reattivo nativo — ogni campo può essere autonomo nel dichiarare la propria validità. È più semplice dell'hook React, non più complesso.

**Scope M2**:

- `ForgeValidator` interface
- `Validators` factory con `required()`, `minLength()`, `maxLength()`, `email()`, `custom(predicate, message)`
- Touch tracking sul campo (errore visibile solo dopo blur)
- `fieldState` aggiornato automaticamente dai validators

---

### [ForgeUI-VAL-002] `ForgeComboBox.isClearButtonSuppressed()` — nullabilità e obbligatorietà

**Contesto**: in M2 `ForgeComboBox` estenderà `ForgeClearableInput`. Il clear button deve essere nascosto quando nessun elemento è selezionato (`selectedItem == null`) o quando il campo è obbligatorio e deve mantenere sempre una selezione.

**Decisione**: `ForgeComboBoxSkin.isClearButtonSuppressed()` restituirà `true` quando:

1. Nessun elemento è selezionato (`selectedItem == null`)
2. Il dropdown è aperto (interazione in corso)
3. Il campo è `required=true` (opzionale, da valutare in M2)

**Stato**: pianificato per M2, da dettagliare in GRM M2.

---

## ForgeBook M3+

---

### [ForgeUI-BOOK-001] ForgeBook — Storybook equivalent per JavaFX

**Contesto**: non esiste uno strumento di catalog e test visuale per componenti JavaFX equivalente a Storybook nel mondo web. Il forge-showcase attuale è una form statica, non un catalog navigabile.

**Decisione**: sviluppare ForgeBook come progetto parallelo a ForgeUI a partire da M3.

**Caratteristiche pianificate**:

- Annotazione `@ForgeStory` per dichiarare i test case direttamente nel codice
- Sidebar navigabile con `ForgeAccordion` (M2) per la gerarchia componenti/storie
- Widget nativi per cambiare tema e lingua senza ricompilazione
- Accessibility checker integrato — testa `AccessibleRole` e `accessibleText` su ogni story
- Indipendente da ForgeUI — può testare qualsiasi componente JavaFX

**Motivazione**: il mercato degli strumenti di test UI per JavaFX è scoperto. Scenic View ispeziona il scene graph ma non è un catalog. TestFX testa ma non mostra. Scene Builder edita ma non documenta. Lo spazio per uno standard de facto esiste.

**Nome**: ForgeBook — richiamo a Storybook (subito riconoscibile a chi del settore) + prefisso Forge (collega all'ecosistema ForgeUI).

**Target M3**: proof of concept con `@ForgeStory`, rendering di 3-4 componenti, sidebar con `ForgeAccordion`, widget tema/lingua.

**Stato**: pianificato per M3, da dettagliare in GRM M3.

---

_DTR aggiornato al termine di Milestone 1 — ForgeUI v1.0.0_