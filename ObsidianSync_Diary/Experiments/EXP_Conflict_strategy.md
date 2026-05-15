# The definitive conflict resolution strategy

## Perché questo file

Prima di riprendere da dove ci eravamo lasciati, facciamo un recap del percorso che ci ha portato fino a qui. 

Nel corso dei test condotti in **Gabriela_Experiment** abbiamo affrontato il caso limite di Gabri che, non spegnendo mai il Mac, potrebbe virtualmente non generare mai gli eventi PUSH_LOGOFF e PULL_LOGON e, lavorando da un altro pc, arriverebbe all'incomoda situazione in cui verrebbe perso l'allineamento con la Source of Truth su GitHub. La soluzione proposta è l'introduzione di un PULL_MANUAL che le consenta di richiedere l'aggiornamento dei file prima di cominciare la sessione lavorativa. Il problema si verifica nel caso in cui Gabri abbia già aperto Obsidian, non sarebbe virtualmente nemmeno necessario modificare nulla: il software tiene traccia del file attualmente aperto dentro .obsidian e basta una semplice navigazione per portare ad un disallineamento.

Claude ha inizialmente proposto una strategia di risoluzione di conflitti semplificata: 
`PULL_MANUAL → hasChanges? → commitLocal + push → pull`
Il test approfondito con git ha subito rilevato che richiedere il push con dei dati disallineati porta ad un errore che richiede l'esecuzione di un merge manuale - necessità di un software per la conflict resolution, idoneo solo ad un utente avanzato.

Si ha quindi ripreso da qui il Grooming al fine di trovare una soluzione bubez-friendly. Claude ha partorito una strategia mirata che prevede di tentare inizialmente il pull e, in caso di fallimento, eseguire uno snapshot del vault dentro una cartella dedicata prima di lanciare `git pull -X theirs`. L'esperienza insegna che fidarsi è bene, non fidarsi è meglio, quindi prima di darla per buona e proseguire nell'analisi dei punti successivi abbiamo eseguito un altro esperimento approfondito: **St.Thomas_skepticism**.

L'outcome di questa procedura è stato incoraggiante ma non ancora risolutivo. Utilizzare `theirs` in un PULL_MANUAL significa piallare il lavoro svolto durante la sessione corrente per aggiornarsi ad una repo già vecchia. Il fatto di disporre di un backup garantisce di essere in grado di ripristinare lo stato con un copia incolla mirato, ma si tratta di una bruttura dal punto di vista funzionale. Oltre a questo, è emerso che concettualmente la strategia `theirs`, corretta per un PULL_LOGON in quanto non vi saranno sessioni aperte, mal si adatta al caso in esame perché porta sistematicamente alla sovrascrittura degli elaborati prodotti prima di richiedere il pull.

Siamo quindi tornati su Claude e, fornendogli l'analisi integrale svolta nel corso dell'esperimento, si è giunti alla strategia basata su `ours` che ora andremo a descrivere e verificare.

## Questa è la volta buona!

L'approccio concordato è simile a quello già verificato, con la differenza che verrà utilizzata la strategia `ours`. Resta da gestire il caso in cui GitHub contenga effettivamente del lavoro inedito che si vuole integrare nel file attuale. Per ottenere questo, si userà un ciclo sui file interessati dal conflitto per estrarre i contenuti online con `git show MERGE_HEAD:<file>` e salvarli in una cartella *remote-conflicts* inserita di fianco a *backup*. Da qui il bubez potrà recuperare le righe da preservare ed eseguire il copia incolla manuale.

[NOTA] L'ultimo intervento manuale è inevitabile perché solo chi ha lavorato sul file locale può sapere quali righe vanno preservate. Non è comunque richiesta alcuna skill tecnica.

La procedura definitiva che andremo a testare, quindi, prevede i seguenti punti:
```
PULL_MANUAL con modifiche locali rilevate
│
├─ git pull
│   │
│   ├─ SUCCESS → fine, nessun backup
│   │
│   └─ CONFLICT (exit code != 0)
│       ├─ git merge --abort (ignorare errore se no merge in corso — confermato dal tuo test)
│       ├─ FIFO backup → backups/Personal_{timestamp}/
│       ├─ git pull -X ours --no-edit
│       ├─ per ogni file in conflitto → git show MERGE_HEAD:<file> → remote-conflicts/<vault-name>_{timestamp}/
│       └─ Toast: "versione locale mantenuta, versione remota disponibile in remote-conflicts"
```
Il Toast visualizzato dall'OS presenterà un pulsante che consente l'apertura diretta della directory dedicata al mantenimento dei file scaricati da remoto; in questo modo agevoleremo ulteriormente le operazioni di allineamento manuale.

[NOTA] Il lavoro in locale viene preservato grazie all'utilizzo di `ours`, il backup FIFO potrebbe non essere più necessario. Seguirà un'analisi dedicata con Claude.

## La simulazione
### 1. Aggiunta del file alla repo su GitHub
Analogamente a come si ha fatto in precedenza, verifichiamo lo stato ed eseguiamo il push su GitHub, nessun problema atteso.
```
PS C:\Users\aless\obsidian-vaults\obsidian-test-vault> git status
On branch main
Your branch is up to date with 'origin/main'.

Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
        modified:   .obsidian/workspace.json
        modified:   Gabriela_experiment.md
        modified:   St.Thomas_skepticism.md

Untracked files:
  (use "git add <file>..." to include in what will be committed)
        Conflict_strategy.md

no changes added to commit (use "git add" and/or "git commit -a")
PS C:\Users\aless\obsidian-vaults\obsidian-test-vault> git add -A
PS C:\Users\aless\obsidian-vaults\obsidian-test-vault> git commit -m "Conflict_strategy start"
[main cda18f0] Conflict_strategy added to repository
 1 file changed, 82 insertions(+)
 create mode 100644 Conflict_strategy.md
PS C:\Users\aless\obsidian-vaults\obsidian-test-vault> git push
Enumerating objects: 4, done.
Counting objects: 100% (4/4), done.
Delta compression using up to 12 threads
Compressing objects: 100% (3/3), done.
Writing objects: 100% (3/3), 3.11 KiB | 3.11 MiB/s, done.
Total 3 (delta 1), reused 0 (delta 0), pack-reused 0 (from 0)
remote: Resolving deltas: 100% (1/1), completed with 1 local object.
To https://github.com/AleDeP10/obsidian-test-vault.git
   a881cba..cda18f0  main -> main
```

### 2. Modifiche divergenti
Stesso trick già applicato negli esperimenti precedenti:
 - file in locale editato con Obsidian
 - accodamento di un testo arbitrario alla versione online, col successivo commit
 A questo punto, il pull iniziale fallirà ed entreremo nel vivo dell'esperimento.

### 3. git pull iniziale
Il pull rileva l'assenza nella repo locale del commit effettuato su GitHub.
```
PS C:\Users\aless\obsidian-vaults\obsidian-test-vault> git pull 
remote: Enumerating objects: 5, done.
remote: Counting objects: 100% (5/5), done.
remote: Compressing objects: 100% (3/3), done.
remote: Total 3 (delta 2), reused 0 (delta 0), pack-reused 0 (from 0)
Unpacking objects: 100% (3/3), 1.33 KiB | 97.00 KiB/s, done.
From https://github.com/AleDeP10/obsidian-test-vault
   cda18f0..31977d6  main       -> origin/main
Updating cda18f0..31977d6
error: Your local changes to the following files would be overwritten by merge:
        Conflict_strategy.md
Please commit your changes or stash them before you merge.
Aborting
```

### 4. git merge --abort
Facciamo abortire la mamma del merge, qualora ve ne fosse bisogno. Ignoriamo l'eventuale errore.
```
PS C:\Users\aless\obsidian-vaults\obsidian-test-vault> git merge --abort
fatal: There is no merge to abort (MERGE_HEAD missing)
```

### 5. FIFO backup
Come accennato sopra, questo procedimento potrebbe non servire: la decisione verrà presa in fase di rifinitura.
Comunque facciamo una copia del file sul Desktop per maggior sicurezza.

### 6. git pull -X ours
Qui ci giochiamo tutto!
```
PS C:\Users\aless\obsidian-vaults\obsidian-test-vault> git pull -X ours
Updating cda18f0..31977d6
error: Your local changes to the following files would be overwritten by merge:
        Conflict_strategy.md
Please commit your changes or stash them before you merge.
Aborting
```
Ok, errore non previsto. Accontentiamo git con un commit locale.
```
PS C:\Users\aless\obsidian-vaults\obsidian-test-vault> git add -A
PS C:\Users\aless\obsidian-vaults\obsidian-test-vault> git commit -m "commit attempt before retrying pull"
[main 98159fa] commit attempt before retrying pull
 1 file changed, 52 insertions(+), 8 deletions(-)
```
Riproviamo e vediamo se devo sbudellare Claude!
```
PS C:\Users\aless\obsidian-vaults\obsidian-test-vault> git pull -X ours --no-edit
warning: in the working copy of 'Conflict_strategy.md', LF will be replaced by CRLF the next time Git touches it
Auto-merging Conflict_strategy.md
Merge made by the 'ort' strategy.
```

[NOTA] Claude, ti è andata bene.

### 7. Scaricamento delle modifiche sul cloud
Questa procedura sarà automatizzata tramite la lettura del log del primo pull, l'estrazione dei file che verrebbero sovrascritti dal merge e l'esecuzione ciclica di `git show MERGE_HEAD:<file>`.
```
PS C:\Users\aless\obsidian-vaults\obsidian-test-vault> git show MERGE_HEAD:Conflict_strategy.md
fatal: invalid object name 'MERGE_HEAD'.
```
... E che è questa roba?

[NOTA] Tiro un paio di bestemmie a Claude e torno con la soluzione, ci siamo quasi!

Dopo una rapida conversazione e qualche martellata, ecco il comando che fa al caso nostro:

```
PS C:\Users\aless\obsidian-vaults\obsidian-test-vault> git --no-pager show FETCH_HEAD:Conflict_strategy.md
```

La console visualizza in questo modo l'intero contenuto del file remoto e... sorpresa :D! Ecco qua che riporto le righe aggiunte su GitHub, che custodivo gelosamente come Easer Egg.

## VICTORYYYY !!!
Se stai leggendo questo paragrafo, vuol dire che l'esperimento si è concluso con un completo successo.

Questo testo è stato infatti introdotto direttamente in un commit performato da GitHub. Si ha seguito la procedura concordata con Claude tramite `git pull -X ours` e si ha salvato lateralmente la versione online del file con `git show MERGE_HEAD:<file>`.

[UPDATE] Il comando giusto usa FETCH_HEAD anziché MERGE_HEAD; occorre far precedere show da --no-pager per fare in modo che venga restituito integralmente il contenuto prelevato dalla repo sul cloud, altrimenti git per default lo aprirebbe con *less* e ciò mal si addice ad un processo automatico.

Fatto questo, accodare la modifica che si desidera preservare con copia-incolla è diventato un gioco da ragazzi.
Il bubez non ringrazia nemmeno, ignaro di tutto il lavoro svolto per semplificargli la vita :p.

## Sommario
La sperimentazione condotta direttamente su git ha richiesto un effort di una giornata e mezza di lavoro, ma ha portato a grandi risultati. In questa situazione, infatti, tutte le dinamiche relative il PULL_MANUAL sono state affrontate nel loro caso peggiore e siamo pronti a mettere mano sul codice di GitService con la certezza che non si presenteranno intoppi lungo il cammino.

Potrebbe valer la pena di fare qualche ragionamento anche sul PUSH_MANUAL: procederò con analizzare i worst-case con Claude e, in presenza di dubbi residui, potremmo svolgere un esperimento affine a quelli condotti per il pull.