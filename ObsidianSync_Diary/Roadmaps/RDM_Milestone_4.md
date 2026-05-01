# Roadmap — Milestone 4

## Obiettivi

| #   | Obiettivo                               | Criterio di accettazione                                                                  |
| --- | --------------------------------------- | ----------------------------------------------------------------------------------------- |
| 1   | Integrazione Windows Task Scheduler     | Tre task configurati: Pull al logon, Push al logoff, Autosave ogni 15 min                 |
| 2   | Integrazione macOS launchd              | Pull plist e Autosave plist caricati; logout script registrato come Login Item            |
| 3   | Validazione end-to-end su vault reale   | Ciclo completo verificato: modifica su PC A → logoff → logon su PC B → modifiche presenti |
| 4   | `config.properties.template` committato | Tutte le chiavi documentate con placeholder; config reali in `.gitignore`                 |
| 5   | Release 1.0.0                           | `git flow release finish 1.0.0` · tag pushato · `main` e `develop` aggiornati             |

---

## Ordine implementativo

1. **Smoke test manuale** — verificare che `mvn package` produca un JAR funzionante; testare `pull`, `push`, `autosave` da riga di comando sul vault reale prima di qualsiasi automazione.

2. **Windows — Task Scheduler** — configurare i tre task seguendo la guida nel README; testare ciascuno manualmente con tasto destro → Esegui; verificare l'output in `obsidiansync.log`.

3. **End-to-end su due macchine Windows** — ciclo completo logoff/logon tra PC A e PC B; questo è il test di accettazione reale del progetto.

4. **macOS — launchd** *(se disponibile un Mac)* — deploy del pull plist e autosave plist; configurare e testare il logout script come Login Item.

5. **`config.properties.template`** — verificare che il template committato contenga tutte le chiavi con placeholder chiari; allinearlo alla superficie di configurazione attuale.

6. **Release 1.0.0** — `git flow release start 1.0.0` · bump versione nel `pom.xml` · `git flow release finish 1.0.0` · `git push origin main develop --tags`.

---

## Rischi

| Rischio                                                                     | Probabilità | Impatto | Mitigazione                                                                                     |
| --------------------------------------------------------------------------- | ----------- | ------- | ----------------------------------------------------------------------------------------------- |
| Il trigger logoff di Task Scheduler scatta prima che il push sia completato | Media       | Alto    | Impostare timeout task a 2 minuti; verificare nella cronologia delle esecuzioni                 |
| EXIT trap macOS non scatta su logout forzato                                | Bassa       | Medio   | Accettato — stessa limitazione di un'interruzione di corrente                                   |
| Percorso JAR con spazi rompe l'invocazione bat                              | Media       | Medio   | Racchiudere tutti i path tra virgolette nei file `.bat`                                         |
| Scadenza del token rompe il push al logoff                                  | Bassa       | Alto    | Usare token fine-grained senza scadenza; loggare gli errori di autenticazione in modo esplicito |