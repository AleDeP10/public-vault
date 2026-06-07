# Troubleshooting Log — ForgeUI M2

Documento ad aggiornamento dinamico. Traccia i problemi incontrati durante lo sviluppo, le cause identificate e le soluzioni adottate. Utile come riferimento rapido e come cheat-sheet per situazioni già risolte.

---

## Git — macchine multiple e credenziali

---

### [TRB-001] Git autentica con l'account sbagliato

**Contesto**: su una macchina di sviluppo secondaria Git è configurato con credenziali diverse da quelle del progetto. Il push viene rifiutato con errore 403.

**Sintomo**:

```
remote: Permission to <owner>/<repo>.git denied to <account>.
fatal: unable to access '...': The requested URL returned error: 403
```

**Causa radice**: Git mantiene una configurazione utente a tre livelli — `system`, `global` (per macchina) e `local` (per repo). Se la configurazione globale usa credenziali errate, tutte le repo sulla macchina le ereditano salvo override locale. Su Windows le credenziali HTTPS sono inoltre memorizzate nel Credential Manager, che può sovrascrivere silenziosamente la configurazione Git.

**Soluzione A — override locale per singola repo (consigliata)**

Sovrascrive le credenziali solo per la repo corrente, lasciando intatto il profilo globale.

```bash
# Dentro la cartella della repo da correggere
git config user.name "tuo-username"
git config user.email "tua-email@esempio.com"

# Verifica
git config user.name
git config user.email

# Mostra tutte le configurazioni attive con la loro origine (local / global / system)
git config --list --show-origin
```

**Soluzione B — override globale**

Sovrascrive per tutte le repo sulla macchina. Adatta solo se la macchina è dedicata esclusivamente a un singolo account GitHub.

```bash
git config --global user.name "tuo-username"
git config --global user.email "tua-email@esempio.com"
```

**Se il problema persiste — Credential Manager Windows**

Le credenziali memorizzate nel Credential Manager possono continuare a sovrascrivere l'autenticazione HTTPS anche dopo aver corretto la config Git.

```
Gestione credenziali → Credenziali Windows → cerca github.com → Rimuovi
```

**Attenzione**: il Credential Manager apre di default la sezione **Credenziali Web**, che non contiene le credenziali Git. GitHub è memorizzato nella sezione **Credenziali Windows** — il riquadro a destra nella schermata iniziale. Cercare `github.com` lì, non nelle Credenziali Web.

Al push successivo Windows chiederà le credenziali corrette.

---

## Git — stato del repository

---

### [TRB-002] `detached HEAD` dopo `git checkout origin/<branch>`

**Contesto**: tentativo di accedere a un branch remoto tramite `git checkout origin/<branch>`. Git entra in stato detached HEAD.

**Sintomo**:

```
Note: switching to 'origin/<branch>'.
You are in 'detached HEAD' state.
HEAD is now at <hash> <messaggio commit>
```

**Causa radice**: `git checkout origin/<branch>` punta HEAD direttamente al commit remoto, non al branch locale corrispondente. I commit fatti in questo stato non appartengono a nessun branch e possono andare persi al prossimo cambio di branch.

Git costruisce un DAG (Directed Acyclic Graph) di commit. HEAD è il puntatore al nodo corrente. In stato normale HEAD punta a un branch, che a sua volta punta a un commit. In detached HEAD, HEAD punta direttamente a un commit — il branch non è coinvolto e i nuovi commit non aggiornano nessun puntatore di branch.

**Soluzione**:

```bash
# Torna al branch main
git switch main

# Se il branch locale non esiste ancora, crealo tracciando il remoto
git switch -c <branch> origin/<branch>
```

**Regola**: usare `git switch <branch>` per spostarsi tra branch locali. `git checkout origin/<branch>` è adatto solo per ispezione in sola lettura.

---

### [TRB-003] `error: remote origin already exists`

**Contesto**: tentativo di aggiungere un nuovo remote `origin` su una repo locale che ne ha già uno configurato.

**Sintomo**:

```
error: remote origin already exists.
```

**Causa radice**: ogni remote deve avere un nome univoco nella repo locale. `git remote add origin <url>` fallisce se il nome `origin` è già occupato, indipendentemente dall'URL destinazione.

**Diagnosi**:

```bash
# Mostra tutti i remote configurati con i rispettivi URL
git remote -v
```

**Soluzione — aggiorna l'URL del remote esistente**:

```bash
git remote set-url origin https://github.com/<owner>/<repo>.git

# Verifica
git remote -v
```

**Soluzione alternativa — rimuovi e ricrea**:

```bash
git remote remove origin
git remote add origin https://github.com/<owner>/<repo>.git
```

---

### [TRB-004] Aggiornamento remote URL su tutte le macchine

**Contesto**: l'URL di una repo remota cambia — per rename, migrazione di owner, o cambio di provider. Ogni macchina di sviluppo mantiene l'URL nella propria configurazione locale e deve essere aggiornata manualmente.

**Causa radice**: Git non ha un meccanismo di propagazione automatica degli URL remoti. Ogni clone locale è indipendente.

**Soluzione**:

```bash
git remote set-url origin https://github.com/<owner>/<nuova-repo>.git
git remote -v   # verifica
```

**Se si vuole ripartire da zero — migrazione completa**

Non consigliata salvo casi eccezionali: si perde l'intera cronologia dei commit.

```bash
git remote remove origin
git remote add origin https://github.com/<owner>/<nuova-repo>.git
git push -u origin main
git push -u origin develop
git push --tags
```

**Nota**: GitHub crea automaticamente un redirect permanente dal vecchio URL al nuovo in caso di rename della repo. I link esistenti nella documentazione continuano a funzionare senza aggiornamenti.