# Surface Cordis design

**Start with [the consolidated plan](consolidation.md)** — high-level application usage, plugin ownership, extraction boundaries, and the demonstration app.

Reviewed 2026-09-08 against latest local/GitHub master at research time:

- Olai: `ce5f233e1b7591d177f72a5041c1f18eacc8a9fb`
- Kolu: `56152455be3d7e4e339d406d95eaa2e72152a2db` (also Olai's pinned Kolu revision)

| Document | Purpose |
|---|---|
| [consolidation.md](consolidation.md) | Current recommended design; proposed APIs are labeled |
| [codex.md](codex.md) | Codex's independent first proposal |
| [fable.md](fable.md) | Fable's independent first proposal |
| [fable-review.md](fable-review.md) | Fable's comparison and review after independence was released |
| [previous-sketch.md](previous-sketch.md) | Superseded HTML text/code archive |

Codex wrote its proposal before reading Fable's. Fable was explicitly instructed not to read the HTML, site or Codex proposal before saving its own. Original proposals are retained so the differences remain reviewable; corrections and decisions belong in the comparison and consolidation.

Research/design only: no framework implementation, commits or PRs. Existing runtime tests were inspected as evidence, not rerun as proof of these proposed APIs. Kolu formatting was run. The Cloudflare Pages project `surface-cordis` was deleted at the user's request; Markdown replaces the local HTML and published page.

Fable endorsed the plan of record and stood down. Its final-check findings were incorporated into consolidation.md: explicit app root/composition bindings, host address validation, component-status caveat, fault-watch and shutdown ordering, per-face exposure, contract identity baseline, header-policy refresh, and the fatal structural-fault limitation. The independent drafts remain unchanged.

## Cordis architecture guide follow-up

The original implementation review remains pinned above. The additional [architecture guide](https://github.com/juspay/olai/blob/139366affae1e8bda6103968ecab9b152559c2f1/docs/architecture/cordis.md) is from Olai #565 (`139366affae1e8bda6103968ecab9b152559c2f1`), newer than the local checkout used for that review.

- [Codex follow-up](codex-cordis-doc-review.md)
- [Fable follow-up](fable-cordis-doc-review.md)

The synthesis now includes ownership below plugin level, atomic/token-owned registration, late-acquisition cleanup, and the existing reconnect guarantees. Proposed composition revisions remain conditional on a demonstrated gap. Original independent designs and the first review are historical; this follow-up updates the consolidated plan.

Fable agreed to the follow-up reconciliation and stood down. Added disposable browser instances, separate host read/control services, canonical rank, and an explicit pin-preservation policy. Composition notifications remain an audit/proof target: gated dispatch must preserve synchronous recomposition and avoid self-disposal deadlock.
