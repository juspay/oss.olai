# The evidence driver — `nix run .#drive`

*Brainstorming, 2026-08-28. Ruled: invocation is `nix run`; lane evidence is a TypeScript section (playwright, full power); sections are throwaway — the PR body carries the section inline in a `<details>` block beside its shots. Not yet dispatched.*

**Premise.** `packages/tests` already owns most of the machinery (`evidence.sh`, `evidence.ts`, `support/serve.sh`, `paneVideo.ts`). What lanes rebuilt by hand this week is what it doesn't speak: a real Claude store under a scratch `HOME` (#417), a worktree-local section (#416, #417), video of a state change (strip-resumed-agent). This PR is those three capabilities behind one `nix run` app.

## Invocation

```bash
# From anywhere in a worktree. No devshell, no `just build-client` first —
# the app carries its environment and builds what's missing.

# Fixture vault + local section
nix run .#drive -- --section ./my-evidence.ts --shots ./shots

# Real Claude store (scratch HOME, cwd rewritten, real packaged adapter)
nix run .#drive -- --store ~/.claude --section ./my-evidence.ts --shots ./shots

# Video instead of stills
nix run .#drive -- --section ./my-evidence.ts --video ./shots/resume.mp4

# No section: stand the app up against real data, poke by hand, Ctrl-C
nix run .#drive -- --store ~/.claude --hold
```

```
nix run .#drive -- [flags]

  --section <file.ts>   worktree-local section module (throwaway)
  --vault <dir>         .olai directory to serve        [default: evidence/fixtures/small]
  --store <dir>         Claude session store; copied to scratch HOME, transcript
                        cwds rewritten to the served vault (the sameDirectory rule)
  --shots <dir>         screenshot destination           [default: ./shots]
  --video <file.mp4>    record instead of stills (paneVideo + CLAUDE.md transcode)
  --hold                serve and wait; no section
  --port <n>            pin a port                       [default: 0, URL via serve.sh]

exit 0  section done, shots written        exit 2  boot failed (full server log)
exit 1  section threw (log tail printed)   always: own processes dead, scratch HOME gone
```

## The PR, as a diff tree

```
 A  evidence/drive.sh              entry: flag parsing, composition, teardown trap
 A  evidence/drive.ts             readiness waits → { page, shot, serverLog } →
                                   dynamic-import of --section (helpers lifted
                                   from packages/tests/evidence.ts)
 A  evidence/store.sh             store copy → scratch HOME, cwd rewrite in every
                                   transcript, adapter env (the #417 plumbing, extracted)
 A  evidence/sections/example.ts  the documented starting point a lane copies
 A  evidence/fixtures/small/      tiny default vault (three outlines, one doc)
 A  evidence/README.md            the section contract, one page
 M  flake.nix                     apps.drive (hunk below)
 M  packages/tests/evidence.sh    boots through evidence/drive.sh; its baked-in
                                   sections become `--section builtin:<name>`
 M  packages/tests/paneVideo.ts   exported; becomes --video's engine
 M  HACKING.md                    "producing evidence" = this, one paragraph
 M  CLAUDE.md                     upload recipe's first line: nix run .#drive
```

```
evidence/
├── drive.sh
├── drive.ts
├── store.sh
├── sections/
│   └── example.ts
├── fixtures/
│   └── small/
│       ├── house.olai
│       ├── roadmap.olai
│       └── notes.md
└── README.md
```

## flake.nix hunk

One app on the existing root flake — not a subflake (a worktree per lane means one lockfile and one `.#` namespace beat per-folder flakes):

```nix
  apps = forAllSystems (pkgs: {
    drive = {
      type = "app";
      program = pkgs.lib.getExe (pkgs.writeShellApplication {
        name = "drive";
        runtimeInputs = [
          pkgs.bun
          pkgs.playwright-driver.browsers   # the same pin the e2e shell uses
          pkgs.ffmpeg                       # --video transcode
        ];
        text = ''
          export PLAYWRIGHT_BROWSERS_PATH=${pkgs.playwright-driver.browsers}
          exec bash ${./evidence/drive.sh} "$@"
        '';
      });
    };
  });
```

## The section contract

Default-export one async function; `Drive` hands it a `page` already past readiness (server bound, client hydrated, agent list settled — the waits #417 hand-hardened, now the driver's), `shot(name)`, and `serverLog()`. The strip-resumed-agent lane's evidence, whole:

```ts
// my-evidence.ts — dismiss × on a quiet subagent, resume it, see it return
import type { Drive } from "./evidence/drive.ts"

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

## UX flow

Author, happy path:

```
$ nix run .#drive -- --store ~/.claude --section ./my-evidence.ts --shots ./shots
drive: client → built (cached)
drive: store  → /tmp/drive-h4Xk/home/.claude    (33 sessions, cwd → /tmp/drive-h4Xk/vault)
drive: server → http://127.0.0.1:43117           (up 2.1s, hydrated)
drive: shot   → shots/my-evidence.strip-before-dismiss.png
drive: shot   → shots/my-evidence.strip-after-dismiss.png
drive: shot   → shots/my-evidence.strip-resumed-returns.png
drive: clean  (3 shots, 11.4s)
```

Author, failure — loud and situated:

```
drive: FAIL in section at shot "strip-resumed-returns" — TimeoutError: waitFor …
drive: server log tail ↓
  [chat] resume pr-author: session a3f… reopened
  [strip] membership: pr-author dismissed=true    ← the bug, visible
drive: kept 2 shots in ./shots; exit 1
```

Orchestrator: briefs shrink to *"Evidence via `nix run .#drive`; section inline in the PR body beside its shots"* — and any lane's evidence is reproducible by copy-pasting the section from the PR. Human: invocation 4 (`--hold`) is the one-command "run olai against my real data."

---

Tier by the roster rule: **ordinary** → grok authors, claude-opus reviews.
