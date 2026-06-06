# `synchronized` — ripasso completo

### Il problema senza sincronizzazione

```
Thread A (autosave):         Thread B (logoff):
legge coda — AUTOSAVE presente
                             legge coda — AUTOSAVE presente
rimuove AUTOSAVE
                             rimuove AUTOSAVE  ← elemento già rimosso, no-op
inserisce nuovo AUTOSAVE
                             inserisce nuovo AUTOSAVE  ← duplicato!
```

Risultato: due AUTOSAVE in coda. Il lock garantisce che la sequenza sia **atomica e indivisibile**.

---

### Tre forme di `synchronized`

**1. Metodo d'istanza** — lock su `this`

java

```java
public synchronized void publish(SyncEvent event) {
    // un solo thread alla volta esegue questo metodo
}
```

**2. Blocco esplicito su `this`**

java

```java
public void publish(SyncEvent event) {
    synchronized (this) {
        // solo questa sezione è protetta
        // codice fuori dal blocco è liberamente concorrente
    }
}
```

**3. Blocco su oggetto dedicato** — più flessibile

java

```java
private final Object lock = new Object();

public void publish(SyncEvent event) {
    synchronized (lock) {
        // puoi avere lock diversi per sezioni diverse
    }
}
```