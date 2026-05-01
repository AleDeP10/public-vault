## [TRB-008] `Package statement is not allowed in compact source files`

**Contesto**: dopo un `git flow feature finish` e successivo `stash pop`, IntelliJ segnala
un errore sulla riga del `package` statement in `Main.java`. Il problema persiste dopo
invalidazione cache, reimport Maven e reinstallazione JDK.

**Causa radice**: durante un tentativo di workaround, `Main.java` era stato spostato
temporaneamente nella root di `src/main/java/` e il metodo `main()` era rimasto
dichiarato senza essere incapsulato in una classe. Maven compila ricorsivamente tutti
i `.java` trovati nel source tree — ha trovato questo file "fantasma" e lo ha interpretato
come **compact source file** (feature Java 21+ preview, stabile da Java 25) perché
privo di dichiarazione di classe.

**Cos'è un compact source file**: feature che permette di scrivere programmi Java senza
dichiarazione esplicita di classe o metodo `main`. Introdotta come preview in Java 21,
stabilizzata in Java 25. Un file con solo `void main() { ... }` è valido in Java 25
ma non in Java 21 — da cui l'errore `not supported at language level '21'`.

**Diagnosi**:
```powershell
Get-ChildItem -Path .\src -Filter "Main.java" -Recurse
```
Output atteso: un solo risultato. Due risultati = file duplicato da rimuovere.

**Soluzione**: incapsulare il metodo `main()` dentro `public class Main { ... }`
nel file corretto, e rimuovere il file duplicato dalla root di `src/main/java/`.

**Prevenzione**: non spostare file Java al di fuori del loro package durante
troubleshooting — il rischio di lasciare file orfani è alto. Usare il refactor
di IntelliJ (`Shift+F6`) per spostare file mantenendo package e riferimenti allineati.