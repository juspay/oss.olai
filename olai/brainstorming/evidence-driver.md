# The evidence driver — one scripted way to see the app working

*Brainstorming, 2026-08-28. Prompted by Scott Fryxell's "The Harness Is the Thing" (his `poster-driver`: the product exposes itself to scripts, so evidence is a command rather than a project). Status: not ruled, not dispatched.*

## The problem

Every dispatched lane must prove its change works with a screenshot or video against real data — that is the pipeline's evidence gate, and it is the right gate. But the *plumbing* to produce that evidence is rebuilt from scratch by every lane, and none of it accumulates:

- The superseded-hint lane (pi, 2026-08-28, PR #417) hand-built a dev-server launcher, a readiness probe, and screenshot capture — then spent a visible round hardening it ("the probe clicks while the agent is still starting, and errors escape silently"), and a cleanup pass killing its own strays.
- The mcp-node-reads lane (grok, same day, PR #416) rolled its own exhibit renderer for the before/after comparison.
- The strip-resumed-agent lane (opus, same day) needs dismiss → resume → screenshot choreography and will build it a third way.

Three lanes, one day, three private copies of the same machinery. Each copy is worse than the last lane's final version, because the last lane's version died with its worktree.

## The idea

One entry point in the olai repository that any lane (or the orchestrator, or a human) can run:

```
bin/drive --vault <path> [--store <path>] --steps <steps-file> --shot <out.png>
```

What it does, in order:

1. **Boots the real server** against the given vault/store under a scratch `HOME` — the arrangement PR #417's evidence proved out: real wire paths, the packaged adapter, no fixtures, and the live vault's one-brain lock never contended.
2. **Waits for readiness properly** — the hardening pi wrote by hand (server up, page hydrated, agent list settled), kept this time instead of re-derived.
3. **Runs a small step script** — a plain list of actions: open the chat panel, click the row titled X, wait for selector Y, type Z. Deliberately tiny; anything needing real logic belongs in the e2e suite, not here.
4. **Captures** — themed screenshot(s) or a video segment, named by the step that produced them.
5. **Exits clean** — kills every process it started, removes its scratch HOME, and fails LOUDLY (with the server log tail) rather than hanging when a step can't complete.

## What changes if this exists

- **Briefs shrink.** "Produce evidence via `bin/drive`; your steps file is part of the PR" replaces a paragraph of plumbing expectations — and the steps file itself documents what the evidence shows.
- **Evidence becomes uniform.** Same boot, same readiness discipline, same framing — which makes the orchestrator's frame-verification sharper (differences in the picture are differences in the product, not in the plumbing).
- **The e2e suite gains a real-data harness** as a side effect: the same driver pointed at a fixture store is a cucumber world; pointed at a copy of a real store it is an evidence run. One machine, two fuels.
- **The orchestrator can spot-check.** A one-command reproduction of any lane's evidence, runnable at merge time from the lane's own steps file.

## Open questions (for the ruling)

1. **Where does it live?** `bin/drive` in the olai repo beside the e2e infrastructure it borrows from, or inside `packages/tests` as an exported runner? The e2e suite already owns browser automation — the driver should be a thin new mouth on that machinery, not a second browser stack.
2. **Steps format.** A flat text file of verbs (click/wait/type/shot) is enough for evidence; resist letting it grow conditionals. Is that constraint acceptable, or do some evidence runs genuinely need logic?
3. **Store copying.** #417 copied the developer's real store and had to rewrite `cwd` inside the copy to satisfy the sameDirectory rule — evidence-run plumbing that belongs in the driver (`--store` implies the rewrite), not in each lane.
4. **Video.** Screenshots are cheap; the strip-resumed-agent shape (state changes over time) wants video. Does the driver own the mp4 transcode recipe that currently lives in CLAUDE.md?
5. **Scope fence.** The driver is for SEEING the app, not testing it — no assertions, no exit-code-on-wrong-pixels. Keeping that line is what keeps it small.

## Next step

If ruled worth doing: a roadmap item under the testing/CI family, grok-shaped by the tier rule (infra/test-shaped, ordinary tier), with this document as the brief's core.
