# 📓 Obsidian Portfolio Vault

Questa repository è la **knowledge base pubblica** di un percorso di apprendimento e sviluppo, costruita attorno a [ObsidianSync](https://github.com/AleDeP10/ObsidianSync) — uno strumento Java CLI self-built che automatizza la sincronizzazione Git di un vault Obsidian tra più macchine Windows.

Il vault documenta il ciclo di vita completo del progetto: decisioni architetturali, pianificazione degli sprint, retrospettive di milestone e autovalutazioni — strutturati come un vero workflow Agile.

---

## Perché esiste

La maggior parte dei portfolio mostra il prodotto finito. Questo mostra **il processo**.

Ogni decisione ha una motivazione. Ogni ostacolo ha un post-mortem. Ogni sprint ha un punteggio. L'obiettivo è dimostrare non solo cosa è stato costruito, ma _come_ — il ragionamento, i trade-off e la crescita sprint dopo sprint.

---

## Struttura della repository

```
obsidian-portfolio/
└── ObsidianSync Diary/
    ├── Diagrams/           ← Diagrammi di architettura e interazione tra componenti
    │   └── DGR_Overview.md
    ├── Groomings/          ← Pianificazione sprint + Architecture Decision Records (stile ADR)
    │   ├── GRM_Milestone_1.md
    │   ├── GRM_Milestone_2.md
    │   ├── GRM_Milestone_3.md
    │   └── GRM_Milestone_4.md
    ├── Milestones/         ← Retrospettive di sprint: obiettivi, problemi, cheat-sheet acquisiti
    │   ├── MILESTONE_1.md
    │   ├── MILESTONE_2.md
    │   └── MILESTONE_3.md
    ├── Roadmaps/           ← Pianificazione implementativa per milestone
    │   └── RDM_Milestone_4.md
    ├── Scores/             ← Valutazioni recruiter-style e personal trainer
    │   ├── SCR_Milestone_2.md
    │   └── SCR_Milestone_3.md
    ├── Troubleshooting/    ← Post-mortem su problemi ambientali e di sviluppo
    │   ├── TRB_Milestone_2.md
    │   └── TRB_Milestone_3.md
    └── Tutorials/          ← Cheat-sheet tecnici acquisiti sprint per sprint
        ├── TTR_Milestone_2.md
        └── TTR_Milestone_3.md
```

---

## Il progetto: ObsidianSync

> Uno strumento Java CLI leggero che automatizza la sincronizzazione Git di un vault Obsidian tra più macchine Windows. Attivato al logon/logoff tramite Task Scheduler, gestisce i flussi pull/push, l'autosave differenziale e la risoluzione dei conflitti con audit logging.

**Tech stack**: Java 21 · Maven · Git CLI via ProcessBuilder · Task Scheduler · Windows Batch

**Codice sorgente**: [github.com/AleDeP10/ObsidianSync](https://github.com/AleDeP10/ObsidianSync)

### Punti architetturali salienti

- **Orchestrazione event-driven** — le operazioni vengono pubblicate come eventi prioritizzati su una coda; l'orchestratore li consuma in modo seriale, prevenendo per design i problemi di concorrenza Git
- **Retry con exponential backoff** — i fallimenti di rete innescano fino a 3 tentativi con delay progressivi (30s → 60s → 120s)
- **Autosave differenziale** — committa solo quando `git status --porcelain` rileva modifiche effettive
- **Risoluzione conflitti** — `git pull -X theirs` con audit logging completo; il remoto è la fonte di verità
- **Dependency inversion** — l'interfaccia `NotificationHook` disaccoppia l'orchestratore da qualsiasi implementazione futura (tray icon, toast, ecc.)

---

## Come leggere questo vault

**Inizia con** `Groomings/GRM_Milestone_1.md` — le scelte architetturali fatte prima di scrivere una riga di codice. Ogni trade-off è documentato con contesto, motivazione e alternative scartate.

**Poi leggi** `Milestones/MILESTONE_1.md` — la retrospettiva: cosa è stato costruito, cosa si è rotto e cosa si è imparato. Seguita da `MILESTONE_2.md` per lo sprint sull'architettura event-driven e `MILESTONE_3.md` per la suite di test.

**Prosegui con** `Groomings/GRM_Milestone_2.md` — come è stato pianificato lo sprint successivo: rischi identificati, priorità stabilite, incognite affrontate prima di toccare la tastiera.

**Quando qualcosa si rompe**, `Troubleshooting/` mostra come è stato diagnosticato e risolto — dai conflitti Git alla misconfiguration del JDK e alla corruzione della cache di IntelliJ.

**Per la panoramica visiva**, `Diagrams/DGR_Overview.md` contiene il diagramma completo delle interazioni tra componenti con ruoli codificati a colori.

**Per la pianificazione implementativa**, `Roadmaps/` contiene l'ordine di delivery suggerito per ogni milestone con analisi dei rischi.

---

## About

**Alessandro De Prato** · Full Stack Developer

[Portfolio](https://aledep10.github.io/) · [Portfolio Vault](https://github.com/AleDeP10/obsidian-portfolio) · [GitHub](https://github.com/AleDeP10) · [LinkedIn](https://www.linkedin.com/in/alessandro-de-prato)