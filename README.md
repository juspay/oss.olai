# oss.olai

This repository is the working memory of an AI agent orchestrator — the "brain" it boots from, kept as plain text files under git.

[Olai](https://github.com/juspay/olai) is an outliner: it serves a directory of `.olai` outline files (trees of nodes with marks, notes, and properties) and the markdown documents beside them. The orchestrator is a long-running AI agent that dispatches coding agents (Claude, Grok, pi) into [kolu](https://github.com/juspay/kolu) terminals, supervises their pull requests through review, CI, and merge — with evidence produced by [saatchi](https://github.com/juspay/saatchi) — and records everything it decides on this board. A fresh orchestrator session reads this repository and knows everything the previous session knew — nothing lives only in a chat transcript.

One vault per orchestrator, one folder per project.

## Layout

- **`orchestrator/`** — how the orchestrator works, written as outlines it re-reads every session:
  - `orchestrator.olai` — the discipline: supervision rules, CI venues, memory rules, the ban on unapproved process changes.
  - `agents.olai` — the roster: which agents can be dispatched, their exact launch commands, quirks, and the reviewer contract.
  - `dag.olai` — the pipeline template a dispatched task walks: implement → review → address findings → CI → evidence → merge → retire.
  - `reminders.olai` — dated reminders that survive restarts.
  - `lanes.olai` — the live day board (deliberately untracked; the day's state is working material, the durable story lives in the roadmap).
- **`projects/`** — one folder per project this orchestrator runs, each linking its repository:
  - `projects/olai/` → [github.com/juspay/olai](https://github.com/juspay/olai) — the outliner itself: `roadmap/` (features, bugs, infra — every work item becomes a PR-shaped node here), `brainstorming/`, `RCA/` (post-mortems), `lowy-electricity/` (architecture-debate conclusions), `prototypes/` (rendered UI mockups rulings were made from).
  - `projects/saatchi/` → [github.com/juspay/saatchi](https://github.com/juspay/saatchi) — the evidence tool (photograph an app doing the thing): its roadmap.
  - `projects/kolu/` → [github.com/juspay/kolu](https://github.com/juspay/kolu) — the terminal fleet: its roadmap (kolu lanes ride kolu's own /be + /ci process).
  - `projects/nixos-unified-template/` → [github.com/juspay/nixos-unified-template](https://github.com/juspay/nixos-unified-template) — the template strangers copy: its roadmap.
- **`brainstorming/`** — cross-project idea documents (e.g. the design that became saatchi).
- **`_olai/`** — vault internals: property type declarations, the Inbox (whose `Deferrals` node parks PR deferrals, each linking its source PR), trash.
- **`.claude/skills/`** — skills the orchestrator can invoke (e.g. the lowy-electricity architecture debate).
- **`briefs/`, `debates/`** — untracked working material: per-task briefs handed to agents, debate turn files.

## How it changes

The orchestrator writes the board only through olai's own operations (which validate every write and commit through one ledger); humans can edit it in the olai app. The git history is the audit trail — the board's notes carry only the current state of each decision, and `git log` is where the history lives.

This vault was extracted from `juspay/olai`'s `docs/` directory in August 2026, with its full commit history, so the olai repository could hold the product and this repository the process.
