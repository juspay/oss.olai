# The evidence driver — `drive.sh`

*Brainstorming, 2026-08-28. Ruled so far: lane evidence is a **TypeScript section** (evidence.ts style, full playwright); sections are **throwaway** (worktree-local, never committed — the PR carries only the pictures). Not yet dispatched.*

**Premise, one paragraph.** `packages/tests` already owns the machinery: `evidence.sh` (fresh copy + fresh server per section, via the shared `support/serve.sh` boot), `evidence.ts` (playwright sections, one shot per gesture), `doorShots.ts`, `paneVideo.ts`. What lanes rebuilt by hand this week is what it *doesn't* speak: a real Claude session store under a scratch `HOME` (#417), a worktree-local section that isn't one of the baked-in names (#416, #417), video of a state change (strip-resumed-agent). `drive.sh` is those three flags on the existing machinery, not a second driver.

## CLI

```bash
# From a lane's worktree, inside `nix develop .#e2e`, client built:

# 1. Fixture vault (what evidence.sh already does, now with a local section)
bash drive.sh --section ./my-evidence.ts --shots ./shots

# 2. Real Claude store (the #417 shape: scratch HOME, cwd rewritten, real adapter)
bash drive.sh --store ~/.claude --vault fixtures/small --section ./my-evidence.ts --shots ./shots

# 3. Video instead of stills (the strip-resumed-agent shape)
bash drive.sh --section ./my-evidence.ts --video ./shots/resume.mp4

# 4. Just stand the server up and leave it running (poke by hand, then Ctrl-C)
bash drive.sh --store ~/.claude --hold
```

```
drive.sh — stand up the real app against real data, run one evidence section, tear down

  --section <file.ts>   worktree-local section module (throwaway; see contract below)
  --vault <dir>         directory of .olai files to serve   [default: fixtures/small]
  --store <dir>         a Claude session store; copied to a scratch HOME, every
                        transcript's cwd rewritten to the served vault (the
                        sameDirectory rule), adapter pointed at the copy
  --shots <dir>         where screenshots land               [default: ./shots]
  --video <file.mp4>    record the run instead of stills (paneVideo machinery,
                        CLAUDE.md's transcode recipe applied on the way out)
  --hold                no section: serve and wait for Ctrl-C
  --port <n>            pin a port (default: 0, bound URL via serve.sh's file)

exit 0  section completed, shots written
exit 1  section threw — server log tail printed, shots-so-far kept
exit 2  boot failed — full server log printed
Always: processes it started are dead, scratch HOME removed.
```

## The section contract (what a lane writes)

Throwaway file in the worktree, default-exports one async function. The worked example is the strip-resumed-agent lane's evidence, whole:

```ts
// my-evidence.ts — dismiss × on a quiet subagent, resume it, see it return
import type { Drive } from "@olai/tests/drive"   // { page, shot, serverLog }

export default async ({ page, shot }: Drive) => {
  await page.getByRole("button", { name: "agent" }).click()
  await page.getByText("pr-author", { exact: false }).waitFor()
  await shot("strip-before-dismiss")             // card present, quiet

  await page.getByLabel("dismiss pr-author").click()
  await shot("strip-after-dismiss")              // card gone — correct

  await page.getByPlaceholder("ask the agent…").fill("@pr-author also fix the docs")
  await page.keyboard.press("Enter")
  await page.getByText("pr-author", { exact: false }).waitFor({ timeout: 15_000 })
  await shot("strip-resumed-returns")            // THE FIX: the card is back
}
```

`Drive` hands the section a `page` that is already past readiness (server bound, client hydrated, agent list settled — the waits #417 hand-hardened, now inside the driver), a `shot(name)` that screenshots into `--shots` with the section's name prefixed, and `serverLog()` for a section that needs to assert nothing but wants context in its failure output.

## UX flow

**The author (in a lane):**

```
$ bash drive.sh --store ~/.claude --section ./my-evidence.ts --shots ./shots
drive: store  → /tmp/drive-h4Xk/home/.claude   (33 sessions, cwd → /tmp/drive-h4Xk/vault)
drive: server → http://127.0.0.1:43117          (up in 2.1s, hydrated)
drive: shot   → shots/my-evidence.strip-before-dismiss.png
drive: shot   → shots/my-evidence.strip-after-dismiss.png
drive: shot   → shots/my-evidence.strip-resumed-returns.png
drive: clean  (3 shots, 11.4s)
$ # attach per CLAUDE.md's upload recipe; PR body links the shots
```

Failure is loud and situated:

```
drive: FAIL in section at shot "strip-resumed-returns" — TimeoutError: waitFor …
drive: server log tail ↓
  [chat] resume pr-author: session a3f… reopened
  [strip] membership: pr-author dismissed=true   ← the bug, visible in the log
drive: kept 2 shots in ./shots; exit 1
```

**The orchestrator (frame-verification, and briefs):** briefs shrink to one line — *"Evidence via `drive.sh`; paste the section and the shots in the PR body"*. Since sections are throwaway, the PR body carries the section **inline in a `<details>` block** beside its shots: the evidence stays reproducible by copy-paste even after the worktree dies, without adding repo surface.

**The human:** `bash drive.sh --store ~/.claude --hold` is also the one-command "run the app against my real data" that today takes manual setup.

## What actually gets built (delta over what exists)

```
packages/tests/
  drive.sh          new — flag parsing; composes the pieces below; owns teardown
  support/serve.sh  exists — the one spelling of the boot (unchanged)
  support/store.sh  new — copy store → scratch HOME, rewrite cwd in every
                    transcript (the #417 plumbing, extracted), export the env
                    the adapter reads
  drive.ts          new — readiness waits + { page, shot, serverLog }, then
                    dynamic-imports the --section file; ~half extracted from
                    evidence.ts's shared helpers
  paneVideo.ts      exists — becomes --video's engine
  evidence.sh/.ts   exist — unchanged; the baked-in sections keep working;
                    long-term they become sections run through drive.ts
```

Tier by the roster's new rule: **ordinary** (bounded, settled design) → commodity author (grok — test/infra-shaped), claude-opus reviews.
