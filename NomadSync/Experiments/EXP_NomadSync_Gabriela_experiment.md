# The Gabriela experiment

## Perché questo file

Le analisi di usabilità, eseguite con Gemini sul protocollo utente di sincronizzazione e poi rifinite con Claude, hanno fatto emergere uno scenario limite che potrebbe alterare il funzionamento del sistema e portare alla perdita del lavoro non salvato.

## Il problema da analizzare

Gabriela (come me col computer) infatti non spegne mai il suo Mac. Supponiamo che lei abbia qui un'istanza di ObsidianSync ed una su un computer Windows in ufficio, sempre accesi e collegati.

[NOTA] I sistemi operativi hanno la sana abitudine di scollegare l'utente dopo una certa soglia di inattività - questo porterebbe a dei PULL_LOGON e PUSH_LOGOFF. Noi consideriamo il caso peggiore e decidiamo che per qualche ragione (es. pc che funge da server) le impostazioni siano state alterate per impedire il logoff automatico.

In questa situazione ipotetica, si potrebbe registrare la seguente sequenza di eventi:

```
1. Mac (casa) → modifica → NO push ❌ 
2. Windows (ufficio, sempre acceso) → modifica → NO push ❌ 
3. Mac (casa) → modifica → push ✅ 
4. Windows (ufficio) → pull → theirs → lavoro Windows PERSO 💀
```

[NOTA] Il punto 3 è la svolta che fa deragliare il treno: dopo essersi resa conto che il lavoro sui due computer non è allineato, Gabri chiama un PUSH_MANUAL dal Mac che altera lo stato su GitHub. Tornata in ufficio, vorrà scaricarsi gli aggiornamenti e richiederà un PULL_MANUAL. Qui interviene la strategia **theirs** in base alla quale la repo online è la Source of Truth più aggiornata, e scarica gli aggiornamenti dal cloud sovrascrivendo le modifiche apportate i giorni precedenti su Windows.

## La soluzione proposta da Claude

In pratica la procedura di aggiornamento del PULL_MANUAL andrebbe rivista affinché verificasse prima la presenza di modifiche e, in caso affermativo, lanciasse commit e push prima di richiedere il pull.

`PULL_MANUAL → hasChanges? → commitLocal + push → pull`

## La verifica preliminare

Siccome sono come San Tommaso - se non vedo non credo - e mi sforzo di essere pure un po' lungimirante, prima di continuare il GROOMING voglio verificare sul campo la correttezza di questo approccio. Useremo quindi questo stesso file per simulare o punti dello use-case di sopra, facendo finta che questo pc sia il Windows dell'ufficio di Gabri dove il theirs (col protocollo as-is) porterebbe alla perdita di lavoro.

[NOTA] Non è necessario disporre del Mac o di un secondo pc su cui applicare le modifiche concorrenti: a noi interessa solo simularne il cambiamento sulla Source of Truth (come se venisse da un push), e possiamo farlo direttamente da GitHub, aprendo questo file in modifica e salvandolo direttamente sulla repo - simuleremo quindi il punto 3. A questo punto ci troveremo nella situazione descritta del caso ed il pull dello step 4 cancellerebbe le modifiche locali.

Procederemo quindi a simulare il comportamento di git applicandovi il controllo preliminare sulle modifiche ed il push intermedio prima di scaricare le modifiche dal repository.

## La simulazione

### 1. Aggiunta del file alla repo su GitHub
In questo momento il file si trova solo sul pc Windows, serve un commit & push per farlo salire sulla repo - da cui Gabri potrà scaricarlo nel Mac.



=> file caricato su GitHub: 
https://github.com/AleDeP10/obsidian-test-vault/blob/main/Gabriela_experiment.md

[NOTA] Ipotizziamo quindi che Gabri faccia un pull dal suo Mac appena tornata a casa.
### 2. Modifiche offline
A questo  punto parte la simulazione effettiva: supponiamo che Gabri faccia delle modifiche offline al file dal Mac e si dimentichi di eseguire il push.
Tornata in ufficio, applica quindi delle modifiche sulla repo locale Windows, qui sono già simulate: stiamo infatti editando questo file da Obsidian in locale successivamente al push.

```
PS C:\Users\aless\obsidian-vaults\obsidian-test-vault> git status              
On branch main
Your branch is up to date with 'origin/main'.

Untracked files:
  (use "git add <file>..." to include in what will be committed)
        Gabriela_experiment.md

no changes added to commit (use "git add" and/or "git commit -a")

PS C:\Users\aless\obsidian-vaults\obsidian-test-vault> git add .

PS C:\Users\aless\obsidian-vaults\obsidian-test-vault> git commit -m "Gabriela_experiment start"

PS C:\Users\aless\obsidian-vaults\obsidian-test-vault> git push
```

[NOTA] Vi sono anche modifiche apportate a .obsidian/workspace.json, questo file serve ad Obsidian per mantenere traccia del file attualmente aperto e la posizione del puntatore. Nei DTR si è deciso di mantenerlo nella repo di modo che ogni volta che si cambia macchina ci si troverà posizionati esattamente nello stesso punto in cui si stava lavorando la sessione precedente.

[NOTA] Come accennato sopra, Gabriela ha dimenticato di pushare le modifiche effettuate sul Mac: la Source of Truth su GitHub è ferma al commit iniziale - ultima riga presente:
`PS C:\Users\aless\obsidian-vaults\obsidian-test-vault> git push`

Ci troviamo quindi nella seguente situazione:
 - file aggiornato su Windows ed in costante evoluzione
 - evoluzione file parallela su Mac
 - GitHub da aggiornare

### 3. Modifiche sul Mac che verranno poi pushate
A Gabriela si accende la lampadina e si ricorda che eseguire il push potrebbe essere una valida idea. Al termine della sessione di lavoro su Mac, quindi, PUSH_MANUAL.

[NOTA] Simuliamo il suo push direttamente dalla UI di GitHub: apriamo il file in modalità Edit ed aggiungiamo:

[UPDATE] modifiche offline da parte di Gabriela
[UPDATE] modifiche offline da parte di Gabriela
[UPDATE] esecuzione add, commit e push

Di seguito, premiamo il tasto verde in alto "Commit changes..." e introduciamo il messaggio di commit nella modale che si aprirà: "Updates by Gabriela committed and pushed from Mac". Premiamo quindi su "Commit changes" in basso ed abbiamo così simulato le interazioni da Mac.

Nuova situazione:
 - GitHub aggiornato all'ultima versione del Mac
 - il file su Windows contiene dati sporchi (tutto quello che è stato aggiunto dopo il push)

### 4. Aggiornamento su Windows

[NOTA] Ci ho messo un'ora e mezza a scrivere tutto questo, quindi faremo un backup del file sul desktop per prevenire che qualcosa vada storto sovrascrivendo tutto il lavoro svolto!

Qui veniamo davvero al punto della situazione e proviamo la procedura descritta da Claude:
`PULL_MANUAL → hasChanges? → commitLocal + push → pull`

[SPOILER] In questo caso abbiamo apportato modifiche sia sulla macchina Windows locale che sul cloud (simulando il push da Mac) sopra lo STESSO FILE. Checchè ne dica Claude, questa situazione puzza di CONFLICT e dubito fortemente che git me la faccia passare liscia. Comunque proviamo!

#### 4.1 hasChanges - sono presenti aggiornamenti prima del pull?

[NOTA] Qui Claude ha fatto un po' di confusione in quanto GitService prevede due comandi di verifica delle modifiche presenti:
 - `hasChanges` - List.of(gitExecutable, "diff", "--quiet") -> exit code 1 per modifiche unstaged
 - `hasUncommittedChanges` - List.of(gitExecutable, "status", "--porcelain") -> lettura output
 All'inizio avevano finalità differenti, poi si ha deciso che l'orchestrator avrebbe usato solo `hasUncommittedChages` in quanto più compatibile col modello mentale di un utente: per verificare lo stato ho sempre fatto `git status` ed affidare le verifiche ad un comando differente può portare a sorprese impreviste. 

Eseguiamo sulla linea di comando:

```
PS C:\Users\aless\obsidian-vaults\obsidian-test-vault> git status --porcelain 
 M .obsidian/workspace.json
 M Gabriela_experiment.md
```

#### 4.2 Controllo sull'output ricevuto
GitService rivela quindi la presenza di modifiche non committate dal fatto che git ha restituito una stringa non vuota contenente il percorso dei file aggiornati.

[NOTA] Abbiamo due possibili situazioni:
 - output vuoto - nessuna modifica, si può procedere al pull senza rischi
 - file rilevati - occorre add + commit + push prima di eseguire il pull (caso di test)
   
[NOTA] Questo è il punto che non mi torna: fin da niubbo sono stato istruito a fare il pull ad inizio giornata apposta per prevenire i conflitti, la mia impressione è che Claude si sia sbagliato e che git risponderà picche!

#### 4.3 commitLocal - staging delle modifiche sulla repo locale
Quando viene richiesto il commit, GitService si limita ad eseguire add e commit in locale, il push avverrà solo al logoff o su esplicita richiesta dell'utente:
```
List.of(gitExecutable, "add", "-A")
List.of(gitExecutable, "commit", "-m", message)
```

Eseguiamo quindi prima l'aggiunta delle modifiche apportate ai file:

```
PS C:\Users\aless\obsidian-vaults\obsidian-test-vault> git add -A
PS C:\Users\aless\obsidian-vaults\obsidian-test-vault> git status --porcelain
M  .obsidian/workspace.json
M  Gabriela_experiment.md
```

[NOTA] git status con porcelain comprime il normale output di git status per renderlo machine-friendly ed esente dalle localizzazioni di sistema. Il comando restituisce ancora gli stessi file di prima perché sono stati aggiunti allo staging ma non committati.

```
PS C:\Users\aless\obsidian-vaults\obsidian-test-vault> git status            
On branch main
Your branch is up to date with 'origin/main'.

Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
        modified:   .obsidian/workspace.json
        modified:   Gabriela_experiment.md
```

L'output di git status in questo caso rivela che le modifiche sono pronte per il commit.

Adesso eseguiamo il commit locale:

```
PS C:\Users\aless\obsidian-vaults\obsidian-test-vault> git commit -m "Gabriela_experiment - commit from Windows

PS C:\Users\aless\obsidian-vaults\obsidian-test-vault> git status --porcelain
<empty_result>
```

[NOTA] Adesso siamo al momento della verità: vediamo chi ha ragione tra me e Claude!

Proviamo ad eseguire il push e vediamo se tuona.

```
PS C:\Users\aless\obsidian-vaults\obsidian-test-vault> git push 
To https://github.com/AleDeP10/obsidian-test-vault.git
 ! [rejected]        main -> main (fetch first)
error: failed to push some refs to 'https://github.com/AleDeP10/obsidian-test-vault.git'
hint: Updates were rejected because the remote contains work that you do not                                                                                                                                                        
hint: have locally. This is usually caused by another repository pushing to
hint: the same ref. If you want to integrate the remote changes, use
hint: 'git pull' before pushing again.
hint: See the 'Note about fast-forwards' in 'git push --help' for details.
```

[NOTA] Mannaggia a quel niubbo di Claude!! Lo dicevo io che così non poteva funzionare.

[PROPOSTA] Se usassimo uno stash per isolare lo modifiche?

[NOTA] Anche così, però, lo stashPop cercherà di fare un merge e troverà con ogni probabilità un conflitto.