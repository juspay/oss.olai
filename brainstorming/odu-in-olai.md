# Odu in olai — CI becomes a face, the CI agent retires

*Ruled with the human 2026-08-29 (question tool): claude-ci retires fully when this ships; discovery is board-driven; the first PR builds the POLYMORPHIC seam (the Dock's hickey cut), with odu as its second tenant, not an odu-special pane. **Implementor: Claude Opus** (the human's word — and the factor rule agrees: a new seam, no precedent to imitate, cross-cutting; tier design-heavy).*

## The shape: live properties, one seam

The board already does this once. A lane's `terminal` prop stores a bare id — a decision-shaped name, never volatile — and the UI gives that prop a living face: the terminal door, the Dock row, the live pane. The kolu machinery under it (thin padi client, `connect`, cells) subscribes; the board never stores what changes.

Generalize that into the seam it always wanted to be:

```jsonc
// a lane node — props stay names; each declared type brings a FACE
{"id":"native-timing", "custom":{
  "terminal":"c56b6183",                    // type: kolu-terminal → the terminal door (exists today)
  "worktree":".worktrees/native-timing"     // type: path → NEW: if <worktree>/.ci/odu.sock is alive, the CI door
}}
```

**The concept is LIVE PROPERTIES** (the human's name, 2026-08-29): a property whose face updates on its own. One seam, dressings per declared type — and each face keeps its honest name: the terminal DOOR (opens a view), the run MATRIX, the ticking took CHIP (native-timing's pomodoro is the same idea pointed inward — the running thing is the task itself). A third kind of living thing later (a deploy? a saatchi session?) is a new dressing, zero new mechanism. Nothing named for one dressing.

## The odu face, concretely

Odu's coordinator holds the run as live typed state on `.ci/<sha7>/odu.sock` — the same surface framework kolu's padi speaks (`odu attach` and the MCP are its existing clients). Olai adds a **thin odu client** the way it added the thin padi client (orch-p0's exact pattern, upstream in juspay/odu):

- The lane's `ci` step row (or the lane row) wears a quiet CI chip while a run is live: `ci · e2e 2m10s · 8/10 ok`.
- The door opens the run matrix: nodes, durations, ok/red/errored — the `odu attach` view, drawn by olai.
- Run gone (sock dead, run settled) → the chip shows the last verdict from the on-disk record, or nothing. No board writes anywhere.

Discovery is board-driven (ruled): the set of live runs = the lanes' `worktree` props probed for `.ci/odu.sock`. No odu registry, no odu changes for phase 1.

## The phases (the kolu family's discipline)

1. **PR 1 — the seam + the read-only face** (ruled scope): live properties generalized from the Dock/terminal-door mechanism; the odu run matrix as the second dressing; the thin odu client package (upstream). Nothing launches, nothing wakes.
2. **PR 2 — events into the feed**: run settled / first red / errored-infra land in the attention drawer (phase-A feed, existing vocabulary; amber stays infrastructure's register). With the phase-B doorbell, a red WAKES the conversation.
3. **PR 3 — launch as an action + the flake rules**: "run CI for this lane" becomes a surface procedure (the phase-3 action family). Classification goes deterministic: post-odu#93, `errored` = infra and `failed` = red are the machine's own words, and FLAKE = failed-then-passed-on-rerun is derivable from run records. The gates phase (orch-p4) then reads CI-green from odu directly; gh statuses stay the publish path for branch protection.

## What retires (ruled: fully)

claude-ci — the seat born 2026-08-29 09:00 and warm-ruled by 13:21 — sunsets when PR 3 lands: launching is an action, watching is a subscription, classifying is rules, reporting is the feed. Its roster row gets the sunset note then; the warm-watcher law and ci-watcher.md retire with it. (Process edits per THE BAN, at that time.) Until then the watcher keeps every lane.

## Honest edges

- **Two surfaces, one client idiom**: kolu's padi socket is long-lived; odu's is per-run and dies at settle. The face must treat sock-gone as an ordinary state (the last-verdict fallback), not an error — the same lesson the watcher pill's amber taught.
- **Remote venues**: the sock lives in the WORKTREE even when nodes run on kolu-ci-9/petit (the coordinator is local) — board-driven discovery covers today's whole venue set.
- **Non-lane runs** (a human's ad-hoc `odu run`): invisible under board discovery; the registry idea stays parked unless that starts mattering.
