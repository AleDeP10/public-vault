# ForgeUI

> Design system production-grade per JavaFX.

ForgeUI è una libreria open-source di componenti per applicazioni desktop JavaFX. Fornisce una gerarchia di componenti pronti all'uso, tematizzabili e accessibili, con una developer experience modellata sui moderni UI kit web.

---

## Componenti — M1

|Componente|Descrizione|
|---|---|
|`ForgeTextField`|Input testuale a riga singola con label, countdown caratteri, stati di validazione, toggle password, clear button, prefix/suffix|
|`ForgeTextArea`|Input testuale multi-riga con righe configurabili, word wrap e maniglia di ridimensionamento|
|`ForgeButton`|Pulsante stilizzato con quattro varianti, tre dimensioni, stato di caricamento, supporto icona e testo responsivo|

---

## Quick start

**Maven:**

```xml
<dependency>
    <groupId>io.github.aledep10</groupId>
    <artifactId>forge-core</artifactId>
    <version>1.0.0</version>
</dependency>
```

**Java:**

```java
ForgeTextField emailField = new ForgeTextField();
emailField.setLabel("Email");
emailField.setType(TextFieldType.EMAIL);
emailField.setMaxLength(254);
emailField.setRequired(true);

ForgeButton salvaButton = new ForgeButton();
salvaButton.setLabel("Salva");
salvaButton.setAccessibleLabel("Salva modifiche");
salvaButton.setVariant(ButtonVariant.PRIMARY);
```

**FXML:**

```xml
<ForgeTextField label="Email" type="EMAIL" maxLength="254" required="true"/>
<ForgeButton label="Salva" accessibleLabel="Salva modifiche" variant="PRIMARY" size="MD"/>
```

---

## Theming

```java
ThemeManager themeManager = new ThemeManager();
themeManager.loadTheme(scene, Theme.LIGHT);

// CSS esterno per white-labelling
themeManager.loadExternalTheme(scene, Path.of("/myapp/themes/corporate.css"));
```

|Tema|Stato|
|---|---|
|`LIGHT`|Disponibile — M1|
|`DARK`|Pianificato — M2|
|`CUSTOM`|Carica qualsiasi CSS esterno tramite `loadExternalTheme()`|

---

## Configurazione

Posiziona `forge.properties` nella root del classpath. Tutte le chiavi sono opzionali.

```properties
forge.control.responsive=true
forge.control.clearbutton.visibility=ON_CONTENT
forge.textinput.countdown.visible=false
forge.theme.default=LIGHT
```

---

## Requisiti

- Java 21 LTS
- JavaFX 21 LTS

---

## Roadmap

|Milestone|Scope|
|---|---|
|**M1** ✅|`ForgeTextField`, `ForgeTextArea`, `ForgeButton`, motore di theming, sistema skin|
|**M2**|`ForgeAccordion`, `ForgeSwitch`, `ForgeCheckbox`, `ForgeComboBox`, tema dark, framework di validazione|
|**M3**|ForgeBook — catalog visuale di componenti e accessibility checker per JavaFX|
|**M3+**|Date/time picker, slider, color picker, radio button|

---

## Documentazione sviluppatore

Vedi `Docs/ForgeUI_README_DEV.md` per architettura, sistema skin, CSS, guida all'estensione e riferimento API completo.

Tutte le decisioni architetturali sono documentate in `Docs/ForgeUI_DTR.md`.

---

## Licenza

MIT — Copyright © 2026 Alessandro De Prato & Gabriela Belmani