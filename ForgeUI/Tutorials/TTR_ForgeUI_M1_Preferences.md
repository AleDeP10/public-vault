# TTR — Java Preferences API

---

## Il problema che risolve

Un'applicazione desktop ha spesso bisogno di ricordare qualcosa tra una sessione e l'altra: l'ultimo tema selezionato, la dimensione della finestra, la lingua preferita. Le opzioni naive sono scrivere su file o su database — entrambe richiedono gestione del path, permessi, parsing. La Preferences API è la soluzione standard Java: un key-value store persistente, cross-platform, senza file da gestire.

---

## Dove vengono salvati i dati

La Preferences API è un'astrazione — il meccanismo di storage dipende dall'OS:

|OS|Storage|
|---|---|
|Windows|Registry (`HKEY_CURRENT_USER\Software\JavaSoft\Prefs`)|
|Linux/Mac|File `~/.java/.userPrefs/`|

Non devi sapere dove — la API è identica su tutti i sistemi. Questo è il punto.

---

## User root vs System root

Esistono due alberi separati:

**`Preferences.userRoot()`** — preferenze specifiche dell'utente corrente. Ogni utente del sistema ha le sue. È quello che vuoi quasi sempre.

**`Preferences.systemRoot()`** — preferenze globali del sistema, condivise tra tutti gli utenti. Richiede permessi di amministratore in scrittura. Usato raramente — tipicamente per configurazioni di sistema o licenze software.

```java
// User root — preferenze dell'utente corrente
Preferences userPrefs = Preferences.userRoot();

// System root — preferenze globali (richiede admin per write)
Preferences systemPrefs = Preferences.systemRoot();
```

**Errore comune**: usare `systemRoot()` per preferenze utente. I dati vengono scritti nel posto sbagliato e in lettura potresti non trovare quello che hai scritto, o trovare i dati di un altro utente.

---

## Nodi

L'albero delle Preferences è organizzato in nodi, analoghi a cartelle. Ogni nodo ha un path gerarchico con `/` come separatore.

Il modo più pulito per ottenere un nodo scoped alla tua classe è:

```java
Preferences prefs = Preferences.userNodeForPackage(MyClass.class);
```

Questo crea (o apre) un nodo il cui path corrisponde al package della classe — es. `io/github/aledep10/forgeui/theme`. I tuoi dati sono isolati da quelli di altre librerie o applicazioni.

---

## Operazioni base

```java
Preferences prefs = Preferences.userNodeForPackage(ThemeManager.class);

// Scrittura
prefs.put("forge.last.theme", "DARK");
prefs.putInt("window.width", 1024);
prefs.putBoolean("countdown.visible", true);

// Lettura con default (mai null — il default è il fallback)
String theme = prefs.get("forge.last.theme", "LIGHT");
int width    = prefs.getInt("window.width", 800);
boolean show = prefs.getBoolean("countdown.visible", true);

// Cancellazione
prefs.remove("forge.last.theme");

// Cancellazione nodo intero con tutti i suoi valori
prefs.clear();
```

Il secondo argomento di `get()`, `getInt()`, ecc. è sempre il **valore di default** — restituito se la chiave non esiste. Non esiste un modo per distinguere "chiave non trovata" da "chiave trovata con quel valore" senza chiamare `get()` due volte — ma nella pratica non serve mai.

---

## Perché iniettare Preferences invece di usarla direttamente

Considera questo codice:

```java
// Versione NON testabile
public Theme getLastUsed() {
    Preferences prefs = Preferences.userNodeForPackage(ThemeManager.class);
    return Theme.valueOf(prefs.get(PREF_KEY_THEME, Theme.LIGHT.name()));
}
```

Questo codice funziona in produzione ma è **impossibile da testare in isolamento**: ogni test che lo esegue legge e scrive sul registry reale del sistema. I test diventano dipendenti dall'ambiente, si influenzano a vicenda, e lasciano sporcizia nel registry.

La soluzione è **dependency injection** — passare l'istanza di Preferences dall'esterno:

```java
public class ThemeManager {

    private final Preferences preferences;

    // Costruttore di produzione — usa il nodo reale
    public ThemeManager() {
        this(Preferences.userNodeForPackage(ThemeManager.class));
    }

    // Costruttore per i test — accetta qualsiasi istanza
    public ThemeManager(Preferences preferences) {
        this.preferences = preferences;
    }

    public Theme getLastUsed() {
        // Usa this.preferences — in test è il mock, in produzione è il nodo reale
        String name = this.preferences.get(PREF_KEY_THEME, Theme.LIGHT.name());
        return Theme.valueOf(name);
    }
}
```

Nei test:

```java
@Mock
private Preferences preferences;   // Mockito crea un'implementazione fittizia

@BeforeEach
void setUp() {
    themeManager = new ThemeManager(preferences);  // inietta il mock
}

@Test
void getLastUsed_darkSaved_returnsDark() {
    // Arrange — il mock risponde con "DARK" quando interrogato
    when(preferences.get(PREF_KEY_THEME, Theme.LIGHT.name()))
        .thenReturn("DARK");

    // Act
    Theme result = themeManager.getLastUsed();

    // Assert
    assertThat(result).isEqualTo(Theme.DARK);
}
```

Il registry del sistema non viene mai toccato. Il test è deterministico, isolato, ripetibile.

---

## Pattern: constructor chaining

Il doppio costruttore usato in ThemeManager è un pattern standard Java chiamato **constructor chaining** — un costruttore chiama l'altro con `this(...)`:

```java
public ThemeManager() {
    this(Preferences.userNodeForPackage(ThemeManager.class));
    //  ↑ chiama il costruttore sotto
}

public ThemeManager(Preferences preferences) {
    this.preferences = preferences;
    // tutta la logica di inizializzazione sta qui — in un solo posto
}
```

Vantaggi: la logica di inizializzazione è scritta una volta sola; il costruttore pubblico senza argomenti è l'API comoda per il consumer; il costruttore con parametro è l'hook per i test. Stesso pattern applicabile con Clock, Random, o qualsiasi risorsa di sistema che vuoi rendere mockabile.

---

## Cheat sheet rapido

| Operazione                     | Metodo                                          |
| ------------------------------ | ----------------------------------------------- |
| Nodo utente scoped alla classe | `Preferences.userNodeForPackage(MyClass.class)` |
| Nodo utente generico           | `Preferences.userRoot().node("mio/path")`       |
| Leggi stringa con default      | `prefs.get(key, defaultValue)`                  |
| Scrivi stringa                 | `prefs.put(key, value)`                         |
| Leggi int                      | `prefs.getInt(key, defaultInt)`                 |
| Leggi boolean                  | `prefs.getBoolean(key, defaultBool)`            |
| Cancella chiave                | `prefs.remove(key)`                             |
| Cancella nodo intero           | `prefs.clear()`                                 |
| Mai usare per dati utente      | `Preferences.systemRoot()`                      |