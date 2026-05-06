# Roadmap — Milestone 4

## Obiettivi

| # | Obiettivo | Criterio di accettazione |
|---|---|---|
| 1 | Refactoring eccezioni `NetworkException` / `GitException` | whitelist pattern di rete, catch separati in `SyncOrchestrator`, retry solo su `NetworkException` |
| 2 | Smoke test manuale su vault reale | pull, push, autosave verificati da riga di comando |
| 3 | Integrazione Windows Task Scheduler | Tre task configurati: Pull al logon, Push al logoff, Autosave ogni 15 min |
| 4 | Validazione end-to-end su vault reale | Ciclo completo verificato: modifica su PC A → logoff → logon su PC B → modifiche presenti |
| 5 | `config.properties.template` committato | Tutte le chiavi documentate con placeholder; config reali in `.gitignore` |
| 6 | Integrazione macOS launchd *(opzionale)* | Pull plist e Autosave plist caricati; logout script registrato come Login Item |
| 7 | Release 1.0.0 | `git flow release finish 1.0.0` · tag pushato · `main` e `develop` aggiornati |

---

## Ordine implementativo

1. **Refactoring eccezioni** — `NetworkException` e `GitException` in `GitService`;
   catch separati in `SyncOrchestrator`; retry applicato solo agli errori di rete.
   **Priorità alta** — impatta la robustezza prima di andare in produzione su Task Scheduler.

2. **Smoke test manuale** — verificare pull, push, autosave da riga di comando sul vault
   reale. ✅ Completato nella sessione corrente.

3. **Windows — Task Scheduler** — configurare i tre task seguendo la guida nel README;
   testare ciascuno manualmente con tasto destro → Esegui; verificare l'output in
   `obsidiansync.log`.

4. **End-to-end su due macchine Windows** — ciclo completo logoff/logon tra PC A e PC B;
   questo è il test di accettazione reale del progetto.

5. **`config.properties.template`** — verificare che il template committato contenga tutte
   le chiavi con placeholder chiari; allinearlo alla superficie di configurazione attuale.

6. **macOS — launchd** *(se disponibile un Mac)* — deploy del pull plist e autosave plist;
   configurare e testare il logout script come Login Item.

7. **Release 1.0.0** — `git flow release start 1.0.0` · bump versione nel `pom.xml` ·
   `git flow release finish 1.0.0` · `git push origin main develop --tags`.

---

## Rischi

| Rischio | Probabilità | Impatto | Mitigazione |
|---|---|---|---|
| Messaggi stderr Git localizzati in italiano — whitelist non funziona | Media | Alto | testare sulla macchina reale prima di rilasciare |
| Il trigger logoff di Task Scheduler scatta prima che il push sia completato | Media | Alto | impostare timeout task a 2 minuti; verificare nella cronologia delle esecuzioni |
| EXIT trap macOS non scatta su logout forzato | Bassa | Medio | accettato — stessa limitazione di un'interruzione di corrente |
| Percorso JAR con spazi rompe l'invocazione bat | Media | Medio | racchiudere tutti i path tra virgolette nei file `.bat` |
| Scadenza del token rompe il push al logoff | Bassa | Alto | usare token fine-grained senza scadenza; loggare gli errori di autenticazione in modo esplicito |