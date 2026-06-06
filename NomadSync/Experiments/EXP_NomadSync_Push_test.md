# Last experiment: PUSH_MANUAL

## Perché questo file
Se ti sei letto in ordine gli esperimenti condotti per PULL_MANUAL (**Gabriela_experiment**, **St.Thomas_skepticism** e **Conflic_strategy**) allora sai già l'argomento di quest'ultimo test. Qualora non lo avessi fatto, ti consiglio di fare uno sforzo di lettura in quanto tutte le problematiche tecniche sono state lì ampiamente descritte e documentate e, come il famoso compositore Paganini, non amo ripetermi.

L'analisi con Claude ha richiesto un singolo breve prompt in quanto sono stati eviscerati tutti i casi d'uso più improbabili del pull e questo ci ha portato ad una maggior padronanza dei comandi avanzati di git.

Come mia previsione iniziale, vi è un solo caso critico:
 - Gabriela lavora un'ora dall'ufficio
 - Lo scheduler di ObsidianSync triggera degli AUTOSAVE periodici (di default ogni 15 minuti)
 - Gabri stavolta prima di andarsene lasciando il computer acceso si ricorda del PUSH_MANUAL -> quattro commit aggiunti a GitHub
 - Tornata a casa, riprende a lavorare sul file senza effettuare prima un pull
   [NOTA] La VERA Gabriela non è così distratta, ma a noi fa comodo ai fini di narrazione ;)
 - Mezz'ora di lavoro, due AUTOSAVE, ora è il caso di pushare
 - git si lamenta del fatto che nella storia del repository locale mancano quattro commit e si rifiuta di allineare il codice

Nessun problema reale Gabri, sono cose che capitano a tutti i niubbi finché non assimilano che con qualunque tool di versioning è necessario aggiornare la repo locale con un pull prima della sessione di lavoro, dopodiché git sarà mansueto come un agnellino!

## La procedura
Il PUSH_MANUAL diventa quindi un'operazione composta: prima si tenta il push, in caso di fallimento si esegue la stessa procedura di PULL_MANUAL e si riesegue nuovamente push.
```
PUSH_MANUAL
│
├─ git push
│   │
│   ├─ SUCCESS → Toast "Push completato" → fine
│   │
│   └─ REJECTED (exit code != 0, "non-fast-forward")
│       ├─ esegui PULL_MANUAL (con tutta la sua logica conflitti)
│       └─ al termine del pull → git push
│           ├─ SUCCESS → Toast "Sincronizzazione completata"
│           └─ FAILURE → Toast errore + log
```

## La simulazione

### 1. Aggiunta del file alla repo su GitHub
Al solito, il file deve esistere sulla repo per poter eseguire il test.
```
PS C:\Users\aless\obsidian-vaults\obsidian-test-vault> git status --porcelain
 M .obsidian/workspace.json
?? Push_test.md
PS C:\Users\aless\obsidian-vaults\obsidian-test-vault> git add -A
PS C:\Users\aless\obsidian-vaults\obsidian-test-vault> git commit -m "Push_test start"
[main 355f49d] Push_test start
 2 files changed, 56 insertions(+), 3 deletions(-)
 create mode 100644 Push_test.md
PS C:\Users\aless\obsidian-vaults\obsidian-test-vault> git push
Enumerating objects: 8, done.
Counting objects: 100% (8/8), done.
Delta compression using up to 12 threads
Compressing objects: 100% (5/5), done.
Writing objects: 100% (5/5), 1.73 KiB | 1.73 MiB/s, done.
Total 5 (delta 3), reused 0 (delta 0), pack-reused 0 (from 0)
remote: Resolving deltas: 100% (3/3), completed with 3 local objects.
To https://github.com/AleDeP10/obsidian-test-vault.git
   367a340..355f49d  main -> main
```

### 2. Modifiche divergenti
Procediamo simulando le modifiche di Gabri dall'altro pc:
 - file in locale editato con Obsidian
 - accodamento di un testo arbitrario alla versione online, col successivo commit
 - ripetiamo gli stessi passaggi altre due volte
 A questo punto, GitHub è avanti di tre commit. Noi stiamo lavorando sul file in locale e vogliamo far salire queste modifiche.

## 3. Push - primo tentativo
Committiamo e tentiamo il push, botto assicurato!
```
PS C:\Users\aless\obsidian-vaults\obsidian-test-vault> git status --porcelain
 M Push_test.md
PS C:\Users\aless\obsidian-vaults\obsidian-test-vault> git commit -am "local update"         
[main db52d42] local update
 1 file changed, 30 insertions(+), 1 deletion(-)
PS C:\Users\aless\obsidian-vaults\obsidian-test-vault> git push
To https://github.com/AleDeP10/obsidian-test-vault.git
 ! [rejected]        main -> main (fetch first)
error: failed to push some refs to 'https://github.com/AleDeP10/obsidian-test-vault.git'
hint: Updates were rejected because the remote contains work that you do not       hint: have locally. This is usually caused by another repository pushing to
hint: the same ref. If you want to integrate the remote changes, use
hint: 'git pull' before pushing again.
hint: See the 'Note about fast-forwards' in 'git push --help' for details.
```

Facciamo attenzione alla seguente riga:
`! [rejected]        main -> main (fetch first)`
Le problematiche relative al mancato allineamento delle versioni sono distinguibili dagli errori di altra natura per la presenza dei pattern `"non-fast-forward"` oppure `"non-fast-forward"`.

[NOTA] Abbiamo allegramente incartato la repo, ora fallirebbe anche il push.

```
PS C:\Users\aless\obsidian-vaults\obsidian-test-vault> git pull
remote: Enumerating objects: 11, done.
remote: Counting objects: 100% (11/11), done.
remote: Compressing objects: 100% (9/9), done.
remote: Total 9 (delta 6), reused 0 (delta 0), pack-reused 0 (from 0)
Unpacking objects: 100% (9/9), 2.71 KiB | 99.00 KiB/s, done.
From https://github.com/AleDeP10/obsidian-test-vault
   355f49d..bd41f15  main       -> origin/main
error: Your local changes to the following files would be overwritten by merge:
        Push_test.md
Please commit your changes or stash them before you merge.
Aborting
Merge with strategy ort failed
```

### 3. Procedura PULL_MANUAL
Gli step li conosciamo dall'esperimento precedente.
```
PS C:\Users\aless\obsidian-vaults\obsidian-test-vault> git pull -X ours --no-edit
Auto-merging Push_test.md
Merge made by the 'ort' strategy.
```
L'output ci dice che il conflitto riguarda Push_test.md, procediamo col recupere la versione online da console.
```
git --no-pager show FETCH_HEAD:Push_test.md
```

### 4. Copia-incolla manuale 
L'aggiornamento del file è immediato, basta inserire le righe incriminate nella posizione desiderata, a prova del più niubbo dei bubez.

Riporto per completezza di seguito le modifiche apportate nei tre commit diretti su GitHub.

[UPDATE] modifica 1

[UPDATE] modifica 2

[UPDATE] modifica 3

[NOTA] Questo step merita delle attenzioni: il bubez deve essere reso consapevole del verificarsi di un conflitto durante il pull automatico tramite un opportuno Toast.

### 5. Esecuzione consistente del push
Adesso git sarà felice di soddisfare la nostra richiesta
```
PS C:\Users\aless\obsidian-vaults\obsidian-test-vault> git status --porcelain
 M Push_test.md
PS C:\Users\aless\obsidian-vaults\obsidian-test-vault> git commit -am "Push_test end"
[main 73c21db] Push_test end
 1 file changed, 64 insertions(+)
PS C:\Users\aless\obsidian-vaults\obsidian-test-vault> git push
Enumerating objects: 11, done.
Counting objects: 100% (11/11), done.
Delta compression using up to 12 threads
Compressing objects: 100% (7/7), done.
Writing objects: 100% (7/7), 3.44 KiB | 3.44 MiB/s, done.
Total 7 (delta 4), reused 0 (delta 0), pack-reused 0 (from 0)
remote: Resolving deltas: 100% (4/4), completed with 1 local object.
To https://github.com/AleDeP10/obsidian-test-vault.git
   bd41f15..73c21db  main -> mai
```

## Sommario
Questo esperimento ha fatto tesoro dell'esperienza fatta nel corso dei test precedenti, le operazioni sono state eseguite senza nessun imprevisto dimostrando la maturità del protocollo di sincronizzazione concordato.