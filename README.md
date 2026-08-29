# oss.olai

The working memory of an AI agent orchestrator — the "brain" it boots from, as plain text under git.

[Olai](https://github.com/juspay/olai) serves this directory as an outliner; the orchestrator is a long-running agent living in olai's chat panel. It reads this repository at boot and knows everything the previous session knew — nothing lives only in a chat transcript.

[**Demo video**](https://x.com/sridca/status/2093354491965735091) — olai working on this very repository.

## The orchestration loop

```mermaid
flowchart TD
    H([👤 human]) <-- rulings · merge words --> O["orchestrator<br/>(lives in olai's chat panel)"]
    O <-- reads the rules · writes the decisions --> V[("this vault<br/>roadmaps · rules · deferrals")]
    O -- briefs a lane --> A["author agent · kolu terminal<br/>claude / grok / pi"]
    A -- opens --> PR[pull request]
    A -- films & photographs via --> S[saatchi]
    S -- evidence on the PR --> PR
    O -- dispatches --> R["reviewer agents · splits<br/>one comment: MUST / SHOULD / NIT"]
    R --> PR
    PR -- reviews folded · CI green from GitHub --> M{"merge<br/>on the human's word"}
    M -- board swept · terminals retired --> O

    classDef olai fill:#d7f0ea,stroke:#2a6058,color:#0b3b34
    classDef kolu fill:#fdf0d5,stroke:#c9a227,color:#5c4a00
    classDef tool fill:#ece6fa,stroke:#8a6ad9,color:#3d2b73
    class O,V olai
    class A,R kolu
    class S tool
```

Every lane walks the same pipeline: **implement → refactor → review → fold → CI → evidence → merge → retire**. Reviews read code (never run suites); CI verdicts come from GitHub, never from an agent's claim; evidence is photographed against real data; deferrals park in the Inbox with a link to the PR that owes them, and merge only on the human's word.

## Layout

| Path | What it is |
|---|---|
| `orchestrator/` | The rules the orchestrator re-reads every boot: discipline, agent roster, pipeline template, reminders |
| `projects/` | One folder per project — roadmaps, brainstorms, RCAs, prototypes:<ul><li>[juspay/olai](https://github.com/juspay/olai) — the outliner</li><li>[juspay/saatchi](https://github.com/juspay/saatchi) — the evidence tool</li><li>[juspay/kolu](https://github.com/juspay/kolu) — the terminal fleet</li><li>[juspay/nixos-unified-template](https://github.com/juspay/nixos-unified-template) — the template</li></ul> |
| `brainstorming/` | Cross-project idea documents (e.g. the design that became saatchi) |
| `_olai/` | Vault internals: property types, the Inbox (deferrals park here, each linking its source PR), trash |
| `briefs/`, `debates/` | Untracked working material |

## How it changes

The orchestrator writes only through olai's own operations — every write validated, every beat one commit; `git log` is the history, the notes carry only the current state. Extracted from `juspay/olai`'s `docs/` in August 2026 with full history.
