# The St. Thomas skepticism

## Perché questo file

"The Gabriela experiment" ha portato alla luce, come previsto, delle falle nel primo sistema di risoluzione dei conflitti proposto da Claude: se la repo contiene commit ancora non scaricati sulla macchina corrente e si modifica gli stessi file da essi coinvolti, git impedirà sia pull che push fino a quando non verrà eseguito il merge.
E' escluso che a un bubez - utente privo di tech skills - venga imposto di imparare ad usare un conflict solver perché lascia il pc sempre acceso dimenticandosi di allineare la repo. Dopo una lunga chat con Claude si è giunti ad una soluzione di compromesso tra semplicità lato utente e ripristino agevolato in caso di errori imprevisti.
In questo file procederemo a testare l'approccio concordato in base alla dottrina che rese celebre il più scettico dei Santi, il "se non vedo non credo" di San Tommaso.

## La nuova procedura

Di seguito è schematizzato il processo di gestione del pull in presenza di conflitti rilevati:

```
PULL_MANUAL con modifiche locali rilevate
│
├─ git pull
│   │
│   ├─ SUCCESS (fast-forward o merge pulito)
│   │   └─ nessun backup, nessun toast → fine
│   │
│   └─ CONFLICT (exit code != 0, file in stato conflitto)
│       ├─ git merge --abort
│       ├─ gestione backup FIFO
│       ├─ snapshot vault → backups/<vault-name>_{timestamp}/
│       ├─ git pull -X theirs
│       └─ Toast azionabile
```

### Note al backup
Qualora si presentassero dei conflitti, git pull fallirebbe restituendo un exit code diverso da zero e sarà necessario eseguire `git pull -X theirs` (vedi la sezione successiva) al fine di forzare git ad allineare i file alla versione sul cloud. Questa è una scelta coerente nel design in quanto, in un normale flusso di utilizzo, il repository GitHub rappresenta la Source of Truth perché viene triggerato al logoff del sistema operativo per inattività.

Questa procedura d'altro canto sovrascrive il lavoro svolto in locale causando la perdita delle righe coinvolte. Per sopperire alla problematica, si introduce dei backup di sicurezza dell'intero vault in una cartella esterna dedicata. In questo modo l'utente sarà successivamente in grado di aprire così preservato ed allineare manualmente la copia su Obsidian.

Si ha optato per una gestione FIFO del backup: si prevede al massimo tre copie per vault, identificate dal nome seguito da un timestamp, e, una volta raggiunto tale limite si eliminerà la versione più vecchia prima di generare il nuovo snapshot e richiedere a git l'allineamento al repository GitHub.

### Descrizione completa del comando
```
git pull -X theirs
```

| Token       | Ruolo                                                                                                                                                                                                                           |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `git pull`  | Scarica i commit dal remote e li integra nel branch locale tramite merge                                                                                                                                                        |
| `-X theirs` | In caso di conflitto su una o più righe, risolve automaticamente scegliendo la versione remota. Si applica a: righe in conflitto sullo stesso file, file modificato localmente e cancellato dal remoto (vince la cancellazione) |

**Cosa preserva**: tutto il lavoro locale che non è in conflitto con il remoto — righe non toccate, file non toccati, file nuovi creati solo in locale.

**Cosa sovrascrive**: esclusivamente i punti di conflitto — righe dove entrambi i lati hanno apportato modifiche diverse, o file dove un lato ha cancellato e l'altro ha modificato.

**Non è**: una sovrascrittura brutale dell'intero vault. È un merge con risoluzione automatica dei soli punti ambigui.

### Toast azionabile
A seguito dell'operazione di pull, l'utente verrà notificato con un Toast degli esiti ottenuti. Questo potrà essere espanso per leggere la lista dei file sovrascritti / cancellati, sarà inoltre presente un tasto per aprire direttamente la directory dell'ultimo snapshot.

## La simulazione

### 1. Aggiunta del file alla repo su GitHub
Cominciamo con l'allineare il repository sul cloud, prima di tutto è bene controllare l'attuale stato in locale.
 ```
 PS C:\Users\aless\obsidian-vaults\obsidian-test-vault> git status --porcelain
 M .obsidian/workspace.json
 M Gabriela_experiment.md
?? St.Thomas_skepticism.md
 ```
 
 [NOTA] Prima di partire con questo esperimento, si ha effettuato il merge con la versione del repository GitHub per risolvere la confusione creata nel corso di "The Gabriela experiment". In questo modo ci si aspetta che la push funzioni correttamente.

```
PS C:\Users\aless\obsidian-vaults\obsidian-test-vault> git add -A
PS C:\Users\aless\obsidian-vaults\obsidian-test-vault> git commit -m "St.Thomas_skepticism start"
[main 8f550b6] St.Thomas_skepticism start
 3 files changed, 87 insertions(+), 6 deletions(-)
 create mode 100644 St.Thomas_skepticism.md
PS C:\Users\aless\obsidian-vaults\obsidian-test-vault> git push 
Enumerating objects: 10, done.
Counting objects: 100% (10/10), done.
Delta compression using up to 12 threads
Compressing objects: 100% (6/6), done.
Writing objects: 100% (6/6), 2.79 KiB | 2.79 MiB/s, done.
Total 6 (delta 3), reused 0 (delta 0), pack-reused 0 (from 0)
remote: Resolving deltas: 100% (3/3), completed with 3 local objects.
To https://github.com/AleDeP10/obsidian-test-vault.git
   6a8f2b4..8f550b6  main -> main
```

### 2. Modifiche divergenti
Questo file è stato modificato nel momento stesso che vi si ha inserito il log del push, ora simuliamo la modifica effettuata sul Mac di Gabriela accodandovi direttamente da GitHub:

[UPDATE] modifiche offline da parte di Gabriela
[UPDATE] modifiche offline da parte di Gabriela
[UPDATE] esecuzione add, commit e push

Analogamente all'esperimento precedente, salviamo quindi il file tramite un commit su main con messaggio "Updates by Gabriela committed and pushed from Mac".

### 3. git pull - atteso failure
```
PS C:\Users\aless\obsidian-vaults\obsidian-test-vault> git pull
Updating 8f550b6..7995f6d
error: Your local changes to the following files would be overwritten by merge:
        St.Thomas_skepticism.md
Please commit your changes or stash them before you merge.
Aborting
```

Come atteso git interrompe il pull, il relativo comando restituisce un exitCode != 0.

### 4. Annullare il merge
Il primo step di gestione del conflitto prevede di annullare il merge con:
```
PS C:\Users\aless\obsidian-vaults\obsidian-test-vault> git merge --abort
fatal: There is no merge to abort (MERGE_HEAD missing).
```
 
[NOTA] Primo risultato inatteso: non essendoci merge in atto, git dà errore.

[TODO] Indagare su questo risultato. Potrebbe essere utile prevederlo nel flusso per annullare eventuali merge ed ignorare questa classe di errore.

### 5. Gestione backup
Eseguiamo una copia dell'intero vault sul desktop, verrà rimossa ad esperimento completato.

### 6. Ripetizione pull con -X theirs
Prova del nove:
```
PS C:\Users\aless\obsidian-vaults\obsidian-test-vault> git add -A
PS C:\Users\aless\obsidian-vaults\obsidian-test-vault> git commit -m "commit changes before pull"   
[main 7ebdb99] commit changes before pull
 2 files changed, 85 insertions(+), 15 deletions(-)
PS C:\Users\aless\obsidian-vaults\obsidian-test-vault> git pull -X theirs
Auto-merging St.Thomas_skepticism.md
Merge made by the 'ort' strategy.
 St.Thomas_skepticism.md | 65 -------------------------------------------------
 1 file changed, 65 deletions(-)
```

All'esecuzione del `git pull -X theirs` si è aperto Notepad++ per inserire il testo del commit associato al merge.

```
Merge branch 'main' of https://github.com/AleDeP10/obsidian-test-vault
# Please enter a commit message to explain why this merge is necessary,
# especially if it merges an updated upstream into a topic branch.
#
# Lines starting with '#' will be ignored, and an empty message aborts
# the commit.
```

Alla chiusura di Notepad++, il comando risulta eseguito con successo. L'opzione theirs ha fatto in modo che questo file regredesse alla versione sul cloud, fino al punto 2. Naturalmente dispongo del backup per ripristinare lo status quo.

[NOTA] L'apertura di Notepad++ può confondere un bubez, che certo non saprebbe che farsene. Occorrerebbe fare in modo che questa azione diventasse veramente silenziosa.

### Ripristino del file sovrascritto
Apriamo la copia salvata sul desktop e rimettiamo il testo che era stato rimosso.

[NOTA] Quello che si ha eseguito nelle ultime fasi dell'esperimento è un orrore funzionale: è stato necessario aprire il file modificato (l'accesso alla cartella backup verrà semplificato dal Toast azionabile) e ripristinare il lavoro svolto in locale.

[SPOILER] Il bubez si troverà davanti ad interventi manuali e comincerà a tirar giù tutti i Santi del Paradiso, compreso il nosto San Tommaso.

La strategia **theirs** è perfetta per il PULL_LOGON per la quale è stato concepito il suo primo utilizzo - appena loggato non avrò lavoro sporco ed è corretto considerare GitHub come Source of Truth - ma nel caso del PULL_MANUAL io sono già connesso e probabilmente sul file ci ho già lavorato. Non mi sembra corretto dover ripristinare il lavoro svolto dal backup, sarebbe meglio mantenere invariati i file locali su cui ho un conflitto e scaricare (da qualche parte ed in qualche modo) separatamente l'equivalente della repo sul cloud per poi eseguire un copia incolla manuale.

[NOTA] Da qualche parte ci dovremo fermare e lasciare al bubez la responsabilità di allineare le righe in conflitto: è un intervento necessario non delegabile al software stesso. Si vuole però ridurre al minimo le sue interazioni col file system per un'esperienza utente fluida e meno prona ad errori.

Occorre pertanto studiare una strategia per scaricare da GitHub, in una cartella posizionata di fianco a backup, solo i file che hanno incontrato il conflitto.

Questo sarà il [TARGET] della prossima analisi.