# 📓 Obsidian Portfolio Vault

This repository is the **public knowledge base** of an ongoing learning and development journey, built around [ObsidianSync](https://github.com/AleDeP10/ObsidianSync) — a self-built Java CLI tool that automates Git-based synchronization of an Obsidian vault across multiple Windows machines.

The vault documents the full lifecycle of the project: architecture decisions, sprint planning, milestone retrospectives, and scored self-assessments — structured as a real-world Agile workflow.

---

## Why this exists

Most portfolios show the final product. This one shows **the process**.

Every decision has a reason. Every obstacle has a post-mortem. Every sprint has a score. The goal is to demonstrate not just what was built, but _how_ — the thinking, the trade-offs, and the growth across iterations.

---

## Repository structure

```
obsidian-portfolio/
└── ObsidianSync Diary/
    ├── Diagrams/           ← Architecture and component interaction diagrams
    │   └── DGR_Overview.md
    ├── Groomings/          ← Sprint planning + Architecture Decision Records (ADR-style)
    │   ├── GRM_Milestone_1.md
    │   ├── GRM_Milestone_2.md
    │   └── GRM_Milestone_3.md
    ├── Milestones/         ← Sprint retrospectives: objectives, problems, cheat-sheets
    │   ├── MILESTONE_1.md
    │   └── MILESTONE_2.md
    ├── Scores/             ← Recruiter-style assessments and personal trainer evaluations
    │   ├── SCR_Milestone_2.md
    │   └── SCR_Milestone_3.md
    ├── Troubleshooting/    ← Post-mortems for environment and development blockers
    │   ├── TRB_Milestone_2.md
    │   └── TRB_Milestone_3.md
    └── Tutorials/          ← Technical cheat-sheets acquired sprint by sprint
        ├── TTR_Milestone_2.md
        └── TTR_Milestone_3.md
```

---

## The project: ObsidianSync

> A lightweight Java CLI tool that automates Git-based sync of an Obsidian vault across Windows machines. Triggered on logon/logoff via Task Scheduler, it handles pull/push workflows, differential autosave, and conflict resolution with audit logging.

**Tech stack**: Java 21 · Maven · Git CLI via ProcessBuilder · Task Scheduler · Windows Batch

**Source code**: [github.com/AleDeP10/ObsidianSync](https://github.com/AleDeP10/ObsidianSync)

### Architecture highlights

- **Event-driven orchestration** — operations are published as prioritized events to a queue; the orchestrator consumes them serially, preventing Git concurrency issues by design
- **Exponential backoff retry** — network failures trigger up to 3 retries with progressive delays (30s → 60s → 120s)
- **Differential autosave** — commits only when `git status --porcelain` detects actual changes
- **Conflict resolution** — `git pull -X theirs` with full audit logging; remote is source of truth
- **Dependency inversion** — `NotificationHook` interface decouples the orchestrator from any future notification implementation (tray icon, toast, etc.)

---

## How to read this vault

**Start with** `Groomings/GRM_Milestone_1.md` — the architectural choices made before writing a single line of code. Every trade-off is documented with context, motivation and discarded alternatives.

**Then read** `Milestones/MILESTONE_1.md` — the retrospective: what was built, what broke, and what was learned. Followed by `MILESTONE_2.md` for the event-driven architecture sprint.

**Follow with** `Groomings/GRM_Milestone_2.md` — how the next sprint was planned: risks identified, priorities set, unknowns acknowledged before touching the keyboard.

**When something breaks**, `Troubleshooting/` shows how it was diagnosed and fixed — from Git conflicts to JDK misconfiguration and IntelliJ cache corruption.

**For the visual overview**, `Diagrams/DGR_Overview.md` contains the full component interaction diagram with colour-coded roles and annotated data flows.

---

## About

**Alessandro De Prato** · Full Stack Developer

[Portfolio](https://aledep10.github.io/) · [Portfolio Vault](https://github.com/AleDeP10/obsidian-portfolio) · [GitHub](https://github.com/AleDeP10) · [LinkedIn](https://www.linkedin.com/in/alessandro-de-prato)