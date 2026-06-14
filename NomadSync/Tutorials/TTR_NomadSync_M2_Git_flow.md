# Git Flow

## Concetto base

Git Flow è una convenzione sui branch che separa nettamente il lavoro in corso dal codice stabile. I branch hanno ruoli fissi:

|Branch|Ruolo|
|---|---|
|`main`|Solo codice rilasciato e stabile. Taggato ad ogni release.|
|`develop`|Integrazione continua dello sprint. Base per le feature.|
|`feature/*`|Una feature alla volta. Nasce da `develop`, torna su `develop`.|
|`release/*`|Preparazione al rilascio. Bugfix finali, bump versione.|
|`hotfix/*`|Fix urgenti su `main`. Nasce da `main`, torna su `main` e `develop`.|

Per il workflow corrente le branch rilevanti sono `main`, `develop` e `feature/*`.

---

## Installazione

### Windows

Git Flow AVH Edition è **incluso automaticamente da Git for Windows** — nessuna installazione separata necessaria.

```bash
# verifica disponibilità
git flow version
# output atteso: 1.12.3 (AVH Edition)
```

### macOS

Su Mac Git Flow **non è incluso** — va installato tramite Homebrew.

```bash
# 1. Installa Homebrew se non è già presente
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# 2. Installa git-flow
#    Prova prima git-flow-avh, se non disponibile usa git-flow
brew install git-flow-avh || brew install git-flow

# 3. Verifica
git flow version
# output atteso: 1.12.3 (AVH Edition)
```

> **Nota:** su alcune versioni di Homebrew `git-flow-avh` non è disponibile come formula separata. In quel caso `brew install git-flow` installa la versione equivalente e il comando funziona identicamente.

### Linux (Ubuntu/Debian)

```bash
sudo apt-get install git-flow
```

---

## Inizializzazione su repo esistente

`git flow init` si esegue su una repo esistente senza distruggere niente. Legge i branch già presenti e li mappa sui ruoli di Git Flow.

```bash
git flow init
# accetta tutti i default premendo Invio
```

Branch `main` e `develop` già presenti vengono riconosciuti direttamente. Il remote `origin` e la history rimangono intatti.

> **Nota:** se la repo ha modifiche non committate, `git flow init` fallisce. Committare o fare stash prima di procedere:
> 
> ```bash
> git add . && git commit -m "chore: pre git-flow init"
> git flow init
> ```

---

## Flusso feature — utilizzo quotidiano

```bash
# inizio lavoro su una nuova feature
git flow feature start nome-feature

# ... lavori, committi normalmente su feature/nome-feature ...

# feature completata: merge automatico su develop, cancella il branch locale
git flow feature finish nome-feature

# pubblica develop aggiornato
git push origin develop
```

---

## Flusso release — a fine milestone

```bash
# apre una release branch da develop
git flow release start 1.0.0

# aggiornamenti finali: versione nel pom.xml, README, changelog
# poi:
git flow release finish 1.0.0
# merge automatico su main E su develop
# crea il tag v1.0.0 automaticamente

git push origin main develop --tags
```

---

## Flusso hotfix — bug urgente su main

```bash
git flow hotfix start nome-fix

# fix del bug, commit

git flow hotfix finish nome-fix
# merge automatico su main E su develop
# crea tag automaticamente

git push origin main develop --tags
```

---

## Recupero commit accidentali su main

Capita di committare direttamente su `main` dimenticando di staccare la feature. Il push verrà rifiutato dalla branch protection — il remoto è al sicuro. Per recuperare la situazione in locale:

```bash
# 1. Controlla quanti commit hai fatto su main per errore
git log --oneline -5

# 2. Annulla N commit locali mantenendo le modifiche in staging
git reset --soft HEAD~N

# 3. Torna su develop e aggiornalo
git checkout develop
git pull origin develop

# 4. Stacca la feature nel modo corretto
git flow feature start nome-feature

# 5. Porta le modifiche e committa
git add .
git commit -m "feat: descrizione della feature"

# 6. Chiudi la feature e pusha
git flow feature finish nome-feature
git push origin develop
```

---

## Mapping sul workflow Agile

|Evento Scrum|Git Flow|
|---|---|
|Inizio sprint|`git flow feature start <nome>` per ogni obiettivo|
|Commit di lavoro|commit normali sul branch feature|
|Fine feature|`git flow feature finish <nome>`|
|Fine milestone|`git flow release start/finish <versione>`|
|Bug urgente su main|`git flow hotfix start/finish <nome>`|

---

## Naming delle feature

Kebab-case, un branch per obiettivo del grooming:

```
feature/sync-event
feature/sync-event-queue
feature/notification-hook
feature/sync-orchestrator
feature/git-service-extended
```

Il diff risultante sarà pulito e leggibile anche per i recruiter.

---

## Cheat-sheet comandi

```bash
git flow init                              # inizializza su repo esistente
git flow feature start <nome>             # nuovo branch feature da develop
git flow feature finish <nome>            # merge su develop, cancella branch
git flow release start <versione>         # branch di preparazione release
git flow release finish <versione>        # merge su main e develop, tag
git flow hotfix start <nome>              # fix urgente da main
git flow hotfix finish <nome>             # merge su main e develop, tag
git push origin main develop --tags       # push completo post-release/hotfix
```