# GROOMING — ForgeUI M2

---

## Obiettivo della milestone

Introdurre `ForgeAccordion` come componente di primo livello e ristrutturare
il progetto in moduli Maven indipendenti — uno per famiglia di componenti.
Ogni modulo è pubblicabile su Maven Central separatamente. Il dev consumer
installa solo quello che usa.

---

## Struttura moduli — decisione architetturale

### Regola fondamentale

```
forge-core       → zero dipendenze interne ForgeUI
forge-*          → dipende solo da forge-core
                   (eccezioni documentate sotto)
forge-book       → dipende da tutti i moduli che cataloga
forge-showcase   → dipende da tutti i moduli
```

### Criterio per collocare una classe astratta

Una classe astratta va in `forge-core` se almeno due moduli distinti ne hanno
bisogno come superclasse o contratto. Va nel modulo specifico se la usano
solo quel modulo e le sue sottoclassi dirette.

Esempi:
- `ForgeInputControl` → `forge-core` — la usano `forge-inputs`, `forge-selection`,
  `forge-accordion` e tutti i futuri componenti di input
- `ForgeClearableInput` → `forge-core` — la usano `forge-inputs` e `forge-selection`
- `ForgeTextInputBase` → `forge-inputs` — la usano solo `ForgeTextField` e
  `ForgeTextArea`, entrambi nello stesso modulo

### Layout moduli completo

```
forge-core
  config/        ForgeProperties, ForgePropertiesLoader
  css/           ForgePseudoClasses, ForgeStyleClasses
  exception/     ForgeException, ThemeException
  theme/         Theme, ThemeManager
  util/          StringUtil, ValidationUtil
  components/    ForgeInputControl (astratta)
                 ForgeClearableInput (astratta)
  enums/         FieldState, ClearButtonVisibility

forge-inputs                          dipende da forge-core
  components/    ForgeTextInputBase (astratta — solo qui)
                 ForgeTextField
                 ForgeTextArea
  enums/         TextFieldType

forge-button                          dipende da forge-core
  components/    ForgeButton
  enums/         ButtonVariant, ButtonSize

forge-selection                       dipende da forge-core
  components/    ForgeComboBox        (estende ForgeClearableInput)
                 ForgeSwitch          (estende ForgeInputControl)
                 ForgeCheckbox        (estende ForgeInputControl)
                 ForgeRadioButton     (estende ForgeInputControl)
  enums/         SelectionMode

forge-accordion                       dipende da forge-core
  components/    ForgeAccordion
                 ForgeAccordionItem
                 ForgeAccordionHeader
                 ForgeAccordionPanel
  enums/         AccordionVariant
  builder/       AccordionBuilder, AccordionItemBuilder
  metadata/      AccordionMetadata, MetadataKey, MetadataRegistry

forge-validation                      dipende da forge-core
  ForgeValidator (interface)
  Validators (factory)
  ForgeForm
  exception/     ForgeValidationException

forge-book                            dipende da forge-core + tutti i moduli
  @ForgeStory annotation
  ForgeStoryAccordion                 (estende ForgeAccordion)
  ForgeBookMetadata
  StoryRenderer
  DragDropLayoutManager
  AccordionLayoutMemento
  ThemeSwitcherWidget
  LangSwitcherWidget
```

---

## Sprint M2 S1 — Refactoring struttura modulare

**Feature branch**: `git flow feature start m2-refactor-modular-structure`

Questo sprint non scrive una riga di logica nuova. Sposta file esistenti
nei moduli corretti e riconfigura i POM. I test rimangono identici.

### Step 1 — Aggiungere i moduli al parent POM

Nel `pom.xml` della root aggiungere:

```xml
<modules>
    <module>forge-core</module>
    <module>forge-inputs</module>
    <module>forge-button</module>
    <module>forge-accordion</module>   <!-- scheletro vuoto per ora -->
    <module>forge-selection</module>   <!-- scheletro vuoto per ora -->
    <module>forge-validation</module>  <!-- scheletro vuoto per ora -->
    <module>forge-showcase</module>
</modules>
```

Creare le cartelle con il `pom.xml` minimo per ogni nuovo modulo.
Ogni `pom.xml` figlio dichiara `forge-core` come dipendenza.

### Step 2 — Spostare le astratte in forge-core

Da `forge-core/components/` (dove già sono) verificare che ci siano:
- `ForgeInputControl.java`
- `ForgeClearableInput.java`

e che i loro enum di supporto siano nel package `forge-core/enums/`:
- `FieldState.java`
- `ClearButtonVisibility.java`

Se sono ancora in un package generico, aggiornarli. Nessuna modifica al codice.

### Step 3 — Creare forge-inputs e migrare

Struttura target:

```
forge-inputs/src/main/java/io/github/aledep10/forgeui/inputs/
  ForgeTextInputBase.java
  ForgeTextField.java
  ForgeTextArea.java
  enums/TextFieldType.java
  skin/ForgeClearableInputSkin.java
  skin/ForgeTextInputBaseSkin.java
  skin/ForgeTextFieldSkin.java
  skin/ForgeTextAreaSkin.java
```

Il `pom.xml` di `forge-inputs` dichiara:
```xml
<dependency>
    <groupId>io.github.aledep10</groupId>
    <artifactId>forge-core</artifactId>
</dependency>
```

Aggiornare i package nei file spostati se il package path cambia.
Aggiornare gli import in tutti i file che li referenziavano.

### Step 4 — Creare forge-button e migrare

Struttura target:

```
forge-button/src/main/java/io/github/aledep10/forgeui/button/
  ForgeButton.java
  enums/ButtonVariant.java
  enums/ButtonSize.java
  skin/ForgeButtonSkin.java
```

Stesso schema: `pom.xml` dipende da `forge-core`, aggiornare package e import.

### Step 5 — Aggiornare forge-showcase

Il `pom.xml` di `forge-showcase` aggiunge dipendenze a `forge-inputs`
e `forge-button`. Verifica che `forge-core` non importi nulla da questi moduli.

### Step 6 — Spostare i test

I test di `ForgeTextField`, `ForgeTextArea` vanno in `forge-inputs/src/test/`.
I test di `ForgeButton` vanno in `forge-button/src/test/`.
I test di `ForgeInputControl`, `ForgeClearableInput` restano in `forge-core/src/test/`.

### Step 7 — Verifica finale

```bash
mvn clean verify
```

136/136 verde. Nessuna dipendenza circolare.
Se IntelliJ segnala import non risolti, `File → Invalidate Caches → Restart`.

### Chiusura sprint

```bash
git flow feature finish m2-refactor-modular-structure
```

Merge su `develop`. Tag non necessario — non è una release pubblica.

---

## Sprint M2 S2 — ForgeAccordion core

**Feature branch**: `git flow feature start m2-accordion-core`

Implementazione del componente `ForgeAccordion` come modulo standalone.
Nessuna dipendenza da `forge-inputs` o `forge-button` — solo `forge-core`.

### Componenti da implementare

**`ForgeAccordion`** — container che gestisce lo stato open/closed dei pannelli.

Proprietà:
- `items` — lista degli `ForgeAccordionItem` registrati
- `multiExpand` — se `false`, un solo pannello aperto alla volta
- `animated` — transizione height 0 → auto
- `variant` — `AccordionVariant` (DEFAULT, BORDERED, FLUSH)

**`ForgeAccordionItem`** — singolo pannello. Contiene header e panel.

Proprietà:
- `id` — UUID, chiave univoca
- `label` — testo dell'header
- `content` — `Node` JavaFX — composizione libera
- `expanded` — stato aperto/chiuso
- `disabled` — header non cliccabile
- `icon` — nodo icona opzionale a sinistra del label
- `level` — livello gerarchico, calcolato automaticamente

**`ForgeAccordionHeader`** — trigger click. Contiene label + chevron animato.

**`ForgeAccordionPanel`** — contenuto collassabile. Può contenere
qualsiasi `Node` JavaFX, incluso un altro `ForgeAccordion` intero.

### Gerarchia nidificata — come funziona

`ForgeAccordionPanel` può contenere altri `ForgeAccordion` come nodi figli.
La struttura si ripete ricorsivamente:

```
ForgeAccordion (livello 0, variant BORDERED)
└── ForgeAccordionItem
    ├── ForgeAccordionHeader "Controls"
    └── ForgeAccordionPanel
        └── ForgeAccordion (livello 1, variant DEFAULT — assegnata automaticamente)
            └── ForgeAccordionItem
                ├── ForgeAccordionHeader "Button"
                └── ForgeAccordionPanel
                    └── ForgeAccordion (livello 2, variant FLUSH)
```

### Calcolo automatico del livello — opzione B

Ogni `ForgeAccordion` ascolta la propria `parentProperty()`. Quando entra
nella scena, risale la gerarchia e conta gli antenati di tipo `ForgeAccordion`.
Il livello determina la variant di default se non specificata esplicitamente.

```
livello 0 → BORDERED
livello 1 → DEFAULT
livello 2+ → FLUSH
```

La variant può sempre essere sovrascritta esplicitamente dal dev.
La variant manuale è **sticky** — sopravvive agli spostamenti nel drag & drop.

Il listener si auto-ricalcola anche a runtime (drag & drop):

```
CASO 1 — primo attach (oldParent null, newParent non null)
  → calcola livello e applica variant
CASO 2 — detach (newParent null)
  → non fare nulla
CASO 3 — re-attach (entrambi non null)
  → ricalcola livello e applica variant
  → se variant era sticky, mantienila
```

### Skin system

`ForgeAccordionSkin` estende `SkinBase<ForgeAccordion>`.
`ForgeAccordionItemSkin` estende `SkinBase<ForgeAccordionItem>`.

CSS variables da definire in `forge-light.css`:

```css
--forge-accordion-bg
--forge-accordion-header-bg
--forge-accordion-header-bg-hover
--forge-accordion-header-fg
--forge-accordion-header-fg-disabled
--forge-accordion-panel-bg
--forge-accordion-border-color
--forge-accordion-border-radius
--forge-accordion-chevron-color
--forge-accordion-chevron-rotation   /* 0deg → 180deg on expand */
--forge-accordion-transition-duration
```

### Accessibilità

| Elemento | Attributo | Valore |
|---|---|---|
| `ForgeAccordionHeader` | `role` | `button` |
| `ForgeAccordionHeader` | `aria-expanded` | `true` / `false` |
| `ForgeAccordionHeader` | `aria-controls` | id del panel |
| `ForgeAccordionPanel` | `role` | `region` |
| `ForgeAccordionPanel` | `aria-labelledby` | id dell'header |
| item disabled | `aria-disabled` | `true` |

Keyboard navigation:
- `Enter` / `Space` → toggle pannello focalizzato
- `ArrowDown` / `ArrowUp` → navigazione tra header
- `Home` / `End` → primo / ultimo header

### TDD — suite minima

```
AccordionTest
  accordion_defaultState_allItemsClosed
  accordion_clickHeader_togglesPanel
  accordion_singleExpand_closesOtherOnOpen
  accordion_multiExpand_keepsMultipleOpen
  accordion_disabledItem_doesNotToggle
  accordion_nestedAccordion_noStateConflict
  accordion_emptyAccordion_rendersWithoutCrash
  accordion_levelResolution_borderedAtLevel0
  accordion_levelResolution_defaultAtLevel1
  accordion_levelResolution_flushAtLevel2Plus

AccordionItemTest
  accordionItem_defaultState_collapsed
  accordionItem_disabled_headerNotFocusable
  accordionItem_withIcon_rendersIcon
  accordionItem_stickyVariant_survivesReparenting
```

---

## Sprint M2 S3 — Builder + Composite

**Feature branch**: `git flow feature start m2-accordion-builder`

### Builder + Fluent Interface

Due classi: `AccordionBuilder` e `AccordionItemBuilder`.

`AccordionBuilder.builder()` è il punto di ingresso statico.
`.add(label)` crea un `AccordionItemBuilder` figlio e lo registra.
`.build()` produce il `ForgeAccordion` finale.

`AccordionItemBuilder` conosce il proprio livello e il proprio parent builder.
`.add(label)` crea un builder figlio annidato.
`.end()` risale al builder padre.

Il livello è noto al momento della costruzione — il builder lo passa
esplicitamente al nodo, senza bisogno di `resolveLevel()`.

### Composite

`ForgeAccordionNode` come interfaccia comune a `ForgeAccordion` e
`ForgeAccordionItem`. Chi attraversa l'albero non distingue tra nodi e foglie.

Metodi minimi dell'interfaccia:
- `getLabel()`
- `getChildren()`
- `accept(AccordionVisitor visitor)`

Il metodo `accept()` abilita il pattern Visitor — usato in M3 per
l'accessibility checker e il JSON export.

### Tre modi per costruire un accordion

Tutti e tre producono lo stesso `ForgeAccordion` finale.

**Modo 1 — Builder programmatico**:
```java
Accordion sidebar = AccordionBuilder.builder()
    .add("Controls")
        .add("Button").end()
        .add("TextField").end()
    .end()
    .build();
```

**Modo 2 — FXML**:
```xml
<ForgeAccordion variant="BORDERED">
    <ForgeAccordionItem label="Controls">
        <ForgeAccordionItem label="Button"/>
        <ForgeAccordionItem label="TextField"/>
    </ForgeAccordionItem>
</ForgeAccordion>
```

**Modo 3 — JSON** (Sprint M2 S4):
```json
{
  "variant": "BORDERED",
  "items": [
    {
      "label": "Controls",
      "items": [
        { "label": "Button" },
        { "label": "TextField" }
      ]
    }
  ]
}
```

---

## Sprint M2 S4 — Metadata + Registry + JSON

**Feature branch**: `git flow feature start m2-accordion-metadata`

### Due layer di metadati separati

**`AccordionMetadata`** — vive in `forge-accordion`. Metadati strutturali
del componente: livello, variant, stato expanded di default.

**`ForgeBookMetadata`** — vive in `forge-book`. Metadati di annotazione:
story reference, autore, tag, versione. Dipende da `AccordionMetadata`.

Separazione netta: `forge-accordion` non sa nulla di `forge-book`.
`forge-book` conosce `forge-accordion` e lo arricchisce.

### MetadataRegistry

Registry centralizzato che tiene traccia di tutti i file JSON di metadati.
Ogni consumer registra il proprio namespace e il proprio loader all'avvio.
File watcher con `java.nio.file.WatchService` — se un JSON cambia su disco,
il Registry invalida la cache e ricarica senza riavvio.

```
namespace "forge-accordion" → forge-accordion-layout.json
namespace "forge-book"      → forge-book-annotations.json
namespace "spotcast"        → spotcast-custom-metadata.json  (custom consumer)
```

### AccordionJsonLoader

Legge un JSON e ricostruisce l'albero `ForgeAccordion`.
Usato da ForgeBook all'avvio per ripristinare il layout salvato.

---

## Sprint M2 S5 — ForgeBook integration

**Feature branch**: `git flow feature start m2-forgebook-integration`

### ForgeStoryAccordion

Estende `ForgeAccordion`. Aggiunge il concetto di **story** come foglia
dell'albero — un test case visuale con un nome e una `Supplier<Scene>`.

`ForgeStoryItemBuilder` estende `AccordionItemBuilder` e aggiunge il metodo
`.story(name, sceneFactory)` — non presente nel builder base di ForgeUI.

### Decorator: AnnotatedAccordionItem

Un `ForgeAccordionItem` base può essere decorato con `ForgeBookMetadata`
senza modificare la classe base. Il developer sceglie se usare il base o
il decorato. Entrambi implementano `ForgeAccordionNode`.

---

## Sprint M2 S6 — Drag & drop + persistenza

**Feature branch**: `git flow feature start m2-accordion-dragdrop`

### Command pattern

Ogni spostamento di accordion è un `AccordionCommand` — reversibile e loggabile.

```
MoveItemCommand    execute() / undo()
DeleteItemCommand  execute() / undo()
CreateItemCommand  execute() / undo()
```

Undo/redo stack: lista di `AccordionCommand` eseguiti.
Undo scorre la lista al contrario. Redo la riscorre in avanti.

### Memento

Il JSON di layout degli accordion è un `AccordionLayoutMemento`.
Ogni salvataggio è un `capture()`, ogni avvio è un `restore()`.
Ogni drag & drop completato crea un Memento — undo ripristina il precedente.

### WatchService

Il `MetadataRegistry` osserva i file JSON con `WatchService`.
Se il file cambia su disco mentre ForgeBook è aperto, la sidebar si aggiorna
senza riavvio — utile quando due membri del team modificano la struttura
in parallelo e NomadSync sincronizza i file.

---

## Sprint M2 S7 — forge-showcase M2 + accessibilità

**Feature branch**: `git flow feature start m2-showcase`

Aggiornare `forge-showcase` con una sidebar `ForgeAccordion` che replica
la struttura di Storybook — almeno tre livelli, tutte le varianti mostrate,
i tre componenti M1 catalogati come stories.

Test accessibilità manuale con screen reader.
Verifica keyboard navigation completa.

---

## DTR da aprire durante il GROOMING

Queste decisioni vanno discusse e registrate nel DTR prima dell'implementazione:

- **[ForgeUI-INFRA-006]** Struttura moduli Maven — motivazione, alternative scartate
- **[ForgeUI-ARCH-009]** Criterio collocazione classi astratte in forge-core
- **[ForgeUI-ARCH-010]** Calcolo livello accordion — opzione A (factory) vs B (parentProperty)
- **[ForgeUI-ARCH-011]** Variant sticky vs ricalcolata al drag & drop
- **[ForgeUI-ARCH-012]** Separazione AccordionMetadata / ForgeBookMetadata
- **[ForgeUI-BOOK-002]** MetadataRegistry — namespace, WatchService, custom JSON

---

*Documento vivo — aggiornato ad ogni sprint del GROOMING M2*