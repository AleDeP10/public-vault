# GRM — ForgeUI Milestone 1: Scaffolding, ForgeTextField, ForgeButton

---

## Contesto

ForgeUI è un design system Java/JavaFX distribuito come libreria open-source su Maven Central.
Nasce come progetto pilota condiviso: il lead-tracker lo integra come consumer reale fin dal primo sprint, validando per design l'esperienza di integrazione della libreria.
L'obiettivo di lungo periodo è fornire ai developer Java un kit di componenti pronti all'uso, themabili e accessibili, con una developer experience analoga ai moderni UI kit web.

Progetto di riferimento per pattern documentali e di design: [TodoList UI Kit](https://github.com/AleDeP10/TodoList/tree/main/ui-kit)

---

## Obiettivi Milestone 1

| # | Obiettivo | Criterio di accettazione |
|---|---|---|
| 1 | Scaffolding Maven multi-module | Parent POM + `forge-core` + `forge-showcase` + `forge-test` compilano senza errori |
| 2 | `ForgeTextField` | Componente renderizzato con countdown caratteri, label, helperText, stati ERROR/WARNING/SUCCESS |
| 3 | `ForgeButton` | Varianti PRIMARY/SECONDARY/GHOST/DANGER, size SM/MD/LG, loading state, responsività, accessibilità |
| 4 | Theming base | CSS light theme caricato a runtime; architettura pronta per multi-tema |
| 5 | `forge-showcase` | Form dimostrativa con tutti i componenti M1 renderizzati via FXML |
| 6 | Pubblicabilità | Coordinate Maven Central corrette; artefatto SNAPSHOT installabile localmente con `mvn install` |

**Release target**: `0.1.0-SNAPSHOT` → `0.1.0` al termine della milestone.

---

## Struttura Maven

```
forge-ui/                                    ← parent POM (packaging: pom)
├── forge-core/                              ← libreria pubblica → pubblicata su Maven Central
│   └── src/main/
│       ├── java/io/github/aledep10/forgeui/
│       │   ├── components/
│       │   │   ├── ForgeTextField.java
│       │   │   └── ForgeButton.java
│       │   └── theme/
│       │       └── ThemeManager.java
│       └── resources/io/github/aledep10/forgeui/
│           ├── themes/
│           │   ├── forge-light.css
│           │   ├── forge-dark.css          ← scaffolding M1, completo M2
│           │   └── forge-custom-template.css
│           ├── forge.properties
│           └── i18n/
│               ├── messages_it.properties
│               └── messages_en.properties
├── forge-showcase/                          ← app dimostrativa → NON pubblicata su MCR
│   └── src/main/
│       ├── java/io/github/aledep10/forgeshowcase/
│       │   └── ShowcaseApp.java
│       └── resources/io/github/aledep10/forgeshowcase/
│           └── views/
│               └── main-form.fxml
└── forge-test/                              ← test automatizzati TestFX → NON pubblicato su MCR
```

`forge-showcase` dichiara `forge-core` come dipendenza Maven locale, simulando un consumer esterno:
```xml
<dependency>
    <groupId>io.github.aledep10</groupId>
    <artifactId>forge-core</artifactId>
    <version>${project.version}</version>
</dependency>
```

---

## Architettura componenti

### ThemeManager

Caricamento tema a runtime su `Scene`. Persistenza tramite `java.util.prefs.Preferences`. Rilevamento tema di sistema tramite `Platform.getPreferences().getColorScheme()` (JavaFX 21.0.1+) con fallback a LIGHT.

```
ThemeManager
  ├── loadTheme(scene, ThemeName)
  ├── loadExternalTheme(scene, Path)       ← API per temi custom del consumer
  ├── getLastUsed() / saveLastUsed()       ← Preferences API
  └── getSystemDefault()                  ← LIGHT | DARK da OS
```

| Enum | CSS | M1 |
|---|---|---|
| `LIGHT` | `forge-light.css` | ✅ completo |
| `DARK` | `forge-dark.css` | scaffolding |
| `CUSTOM` | path esterno | scaffolding |

### ForgeTextField

Estende `Control` (non `TextField` direttamente) per permettere layout composito con label, countdown e helper. Usa `TextField` internamente come nodo delegato con binding su `textProperty()`.

```
ForgeTextField
  ├── StringProperty value
  ├── StringProperty label
  ├── StringProperty helperText
  ├── IntegerProperty maxLength
  ├── BooleanProperty countdownVisible     ← default da forge.properties
  ├── BooleanProperty responsive           ← default true
  └── ObjectProperty<FieldState> state     ← DEFAULT | ERROR | WARNING | SUCCESS
```

Comportamento: `ChangeListener` su `textProperty()` aggiorna il countdown live. `TextFormatter` blocca input oltre `maxLength`. In layout stretto la label si sposta sopra il campo quando `responsive=true`.

### ForgeButton

Estende `Button`.

```
ForgeButton
  ├── ObjectProperty<ButtonVariant> variant  ← PRIMARY | SECONDARY | GHOST | DANGER
  ├── ObjectProperty<ButtonSize> size        ← SM | MD | LG
  ├── BooleanProperty loading                ← spinner inline, click disabilitato
  ├── BooleanProperty hideTextOnSmall        ← default true, override per istanza
  └── StringProperty accessibleLabel         ← obbligatorio, sempre su setAccessibleText()
```

`GHOST`: pulsante senza sfondo solido, bordo visibile, background trasparente. Al hover: fill semi-trasparente. Usato per azioni secondarie/neutrali (es. "Annulla") che non devono competere visivamente con la PRIMARY.

### forge-showcase — form dimostrativa

Singola `Scene`, nessuna navigazione M1. Contenuto FXML:
- `ForgeTextField` label "Indirizzo", maxLength=200, countdownVisible=true
- `ForgeTextField` label "Città", maxLength=100, state=ERROR, helperText="Campo obbligatorio"
- `ForgeTextField` label "Note", maxLength=500, countdownVisible=false
- `ForgeButton` PRIMARY "Salva" / SECONDARY "Anteprima" / GHOST "Annulla" / DANGER SM "Elimina"
- Toggle Light/Dark/Custom (Dark e Custom disabilitati M1)

---

## Rischi

| Rischio | Probabilità | Impatto | Mitigazione |
|---|---|---|---|
| `getColorScheme()` non disponibile su JavaFX < 21.0.1 | Bassa | Basso | fallback a LIGHT |
| CSS JavaFX non supporta proprietà CSS standard | Alta | Medio | solo `-fx-` properties documentate |
| Namespace `io.github.aledep10` provvisorio | Alta | Basso | migrazione pianificata a brand definitivo, documentata |

---

## Scope rimandato a M2

- Tema DARK completo
- API temi custom (`loadExternalTheme`)
- `ForgeToggleSwitch`, `ForgeComboBox`, `ForgeComboButton`
- `forge-showcase` come catalog navigabile (ListView componenti + panel preview)
- TestFX automatizzati
- Pubblicazione su Maven Central (release stabile `0.1.0`)

---

# Sprint 1 — Scaffolding e componenti base

> Obiettivi: struttura Maven, ThemeManager, ForgeTextField, ForgeButton, forge-showcase.
> Tutti gli obiettivi di M1 ricadono in questo sprint.

## Decisioni

---

### [ForgeUI-M1-S1] Struttura Maven multi-module fin dal primo commit

**Contesto**: il progetto è destinato a tre moduli distinti con responsabilità separate. La struttura Maven non è refactorable in modo indolore dopo i primi commit.
**Decisione**: multi-module Maven con parent POM dal primo commit.
**Motivazione**: `forge-showcase` dipende da `forge-core` come consumer esterno fin dal giorno 1 — valida per design l'esperienza di integrazione senza richiedere un progetto separato fittizio.
**Alternativa scartata**: single-module iniziale con refactor successivo — rimanda il problema, rischia dipendenze circolari.

---

### [ForgeUI-M1-S1] GroupId provvisorio `io.github.aledep10`

**Contesto**: Maven Central richiede un namespace verificabile. Il brand definitivo di ForgeUI non ha ancora un dominio. La spesa per dominio + hosting non è giustificata allo stato attuale delle finanze di progetto.
**Decisione**: `io.github.aledep10` per i primi rilasci SNAPSHOT. Migrazione a namespace brandizzato appena il dominio sarà disponibile.
**Motivazione**: verificabile gratuitamente tramite GitHub. Non blocca la pubblicazione. La migrazione è un'operazione documentata e gestibile su SNAPSHOT — nessun contratto di stabilità dichiarato verso i consumer.

---

### [ForgeUI-M1-S1] JavaFX 21 LTS

**Contesto**: tutti i progetti del portfolio usano Java 21 LTS. JavaFX ha release cadence separata da OpenJDK.
**Decisione**: JavaFX 21 LTS.
**Motivazione**: allineamento con Java 21; stabilità LTS; `Platform.getPreferences().getColorScheme()` disponibile da 21.0.1 per rilevamento tema di sistema.
**Alternativa scartata**: JavaFX 23/24 — nessun LTS, rischio breaking changes.

---

### [ForgeUI-M1-S1] FXML come standard di layout

**Contesto**: i layout possono essere definiti programmaticamente o dichiarativamente in FXML.
**Decisione**: FXML ovunque.
**Motivazione**: separazione netta layout/logica; testabilità indipendente del controller; leggibilità immediata della gerarchia; compatibilità con Scene Builder.
**Alternativa scartata**: layout programmatico puro — verboso, mescola struttura e logica.

---

### [ForgeUI-M1-S1] ForgeTextField come Control composito

**Contesto**: label flottante + countdown + helper text richiedono nodi figli che non sono gestibili estendendo direttamente `TextField`.
**Decisione**: `ForgeTextField` estende `Control`; `TextField` interno come nodo delegato con binding su `textProperty()`.
**Motivazione**: pieno controllo sul layout; pattern usato da ControlsFX e AtlantaFX.
**Alternativa scartata**: estensione diretta di `TextField` — non permette nodi figli senza override non documentati.

---

### [ForgeUI-M1-S1] ThemeManager con Preferences API e fallback

**Contesto**: necessario persistere l'ultimo tema selezionato e rilevare il tema di sistema al primo avvio.
**Decisione**: `java.util.prefs.Preferences` per la persistenza; `Platform.getPreferences().getColorScheme()` con fallback a LIGHT se API non disponibile.
**Motivazione**: cross-platform senza dipendenze esterne; fallback garantisce comportamento prevedibile su qualsiasi JVM.
**API temi custom**: `ThemeManager.loadExternalTheme(scene, path)` — scaffolding M1, implementazione M2. Ispirazione: sistema di template customizzabili del portfolio australiano (Joomla/Oxygen).

---

### [ForgeUI-M1-S1] `accessibleLabel` obbligatorio su ForgeButton

**Contesto**: `hideTextOnSmall=true` nasconde il testo visivo in layout compatto. Un pulsante senza testo visivo è inaccessibile agli screen reader senza alternativa testuale.
**Decisione**: `accessibleLabel` valorizzato sempre su `setAccessibleText()`, indipendente da `hideTextOnSmall`. Il developer consumer può disattivare `hideTextOnSmall` per istanza, ma non può omettere `accessibleLabel`.
**Motivazione**: WCAG 2.1 AA. Pattern analogo ad `aria-label` nel web kit del portfolio TodoList.