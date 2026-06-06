# 📓 Knowledge Base di progetto

Questa repository è la **knowledge base condivisa** dei progetti per cui Alessandro De Prato ricopre il ruolo di Product Owner, tenuta sincronizzata tra più macchine e utenti tramite [NomadSync](https://github.com/AleDeP10/NomadSync) — uno strumento Java CLI self-built che automatizza la sincronizzazione Git di un vault Obsidian su Windows, macOS e Linux.

Il vault documenta il ciclo di vita completo di ogni progetto: decisioni architetturali, pianificazione degli sprint, retrospettive di milestone — strutturati in un workflow Agile.

---

## Perché esiste

La maggior parte dei portfolio mostra il prodotto finito. Questo mostra **il processo**.

Ogni decisione ha una motivazione. Ogni ostacolo ha un post-mortem. L'obiettivo è documentare non solo cosa è stato costruito, ma _come_ — il ragionamento, i trade-off e la crescita sprint dopo sprint.

---

## Progetti

- **NomadSync** — Strumento Java CLI che automatizza la sincronizzazione Git di un vault Obsidian tra più macchine e utenti, attivato al logon/logoff tramite Task Scheduler. Cross-platform: Windows, macOS, Linux.
- **ForgeUI** — Design system production-grade per JavaFX con gerarchia di componenti, architettura skin e motore di temi. Costruito per applicazioni desktop Java.

---

## Struttura del vault

Ogni progetto vive nella propria cartella di primo livello e segue la stessa tassonomia documentale:

```
<NomeProgetto>/
├── <NomeProgetto>_README.md  ← Descrizione generale del progetto
├── Docs/                     ← Documentazione di progetto in italiano
│   ├── <NomeProgetto>_DTR.md         (Decision Track Record consolidato)
│   ├── <NomeProgetto>_README_DEV.md  (Manuale sviluppatore)
│   └── <NomeProgetto>_README_USER.md (Manuale utente)
├── Groomings/        ← Pianificazione sprint e decisioni architetturali
├── Milestones/       ← Retrospettive di sprint: obiettivi, problemi, lezioni apprese
├── Tutorials/        ← Cheat-sheet tecnici acquisiti sprint per sprint
├── Troubleshooting/  ← Post-mortem su problemi ambientali e di sviluppo
├── Diagrams/         ← Diagrammi di architettura e interazione tra componenti
└── Experiments/      ← Spike esplorativi e validazioni
```

Non tutte le cartelle sono presenti in ogni progetto — compaiono solo i tipi di documento effettivamente prodotti per quello sprint.

---

## Come leggere questo vault

**Inizia dal README di progetto** (`<NomeProgetto>_README.md`) per capire cos'è e cosa fa.

**Prosegui con il Grooming** (`GRM_*`) — le decisioni architetturali prese prima di scrivere una riga di codice. Ogni trade-off è documentato con contesto, motivazione e alternative scartate.

**Chiudi con la Milestone** (`MLS_*`) — la retrospettiva di sprint. Obiettivi, risultati, problemi incontrati e cheat-sheet acquisiti.

**Quando qualcosa si rompe**, `Troubleshooting/` mostra come è stato diagnosticato e risolto.

**Per i dettagli tecnici**, `Tutorials/` contiene cheat-sheet autonomi sugli strumenti e i pattern introdotti in ogni sprint. Sono leggibili in isolamento, senza contesto di sprint.

**Per la documentazione completa**, `Docs/` contiene il DTR consolidato, il manuale sviluppatore e il manuale utente di ciascun progetto — equivalenti in italiano della documentazione pubblicata sulle rispettive repository.

Il pattern si ripete e si approfondisce di milestone in milestone:

```
GRM (pianificazione) → MLS (retrospettiva) → TRB (troubleshooting) → TTR (tutorials)
```

---

## Autori

**Alessandro De Prato** · Senior Software Engineer – Tech Lead [Portfolio](https://aledep10.github.io/) · [GitHub](https://github.com/AleDeP10) · [LinkedIn](https://www.linkedin.com/in/alessandro-de-prato)

**Gabriela Belmani** · Software Engineer [GitHub](https://github.com/Belmani) · [LinkedIn](https://www.linkedin.com/in/gabriela-da-sa%C3%BAde-belmani-tumfart)