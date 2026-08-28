# The evidence driver — `nix run ./evidence`

*Brainstorming, 2026-08-28. Ruled: `nix run`; TypeScript sections; sections throwaway (inlined in the PR body beside their shots); **everything lives in `evidence/` with as little as possible leaking to the repo root — the folder is a future standalone repo** (the vault extraction already proved the wholesale-with-history move).*

**Premise.** `packages/tests` has evidence machinery (`evidence.sh`, `serve.sh`, `paneVideo.ts`) but lanes rebuilt plumbing anyway: a real Claude store under a scratch `HOME` (#417), worktree-local sections (#416, #417), video (strip-resumed-agent). This PR builds the self-contained answer — and deliberately does NOT refactor `packages/tests` onto it (that coupling would chain the folder to this repo; it can migrate the day the folder is upstreamed and comes back as an input).

## Invocation

```bash
# Own flake — nothing registered on the root flake. From any worktree:

nix run ./evidence -- ./my-evidence.ts                     # fixture vault, run section
nix run ./evidence -- --store ~/.claude ./my-evidence.ts   # real Claude store
nix run ./evidence -- --store ~/.claude                    # no section = just serve; Ctrl-C
```

```
nix run ./evidence -- [--vault <dir>] [--store <dir>] [section.ts]

  section.ts     worktree-local section module (throwaway). Absent: serve and hold.
  --vault <dir>  .olai directory to serve   [default: evidence/host/fixtures/small]
  --store <dir>  Claude session store; copied to scratch HOME, transcript cwds
                 rewritten to the served vault (the sameDirectory rule)

Not flags, on purpose:
  shots  → always ./shots (the section names each one)
  video  → the section's call: `export const record = true` films the run
  port   → always ephemeral; the bound URL is printed
  hold   → the absence of a section

exit 0  section done          exit 2  boot failed (full server log printed)
exit 1  section threw (log tail printed, shots-so-far kept)
always: own processes dead, scratch HOME removed
```

## The PR, as a diff tree — one `M`, everything else inside the folder

```
 A  evidence/flake.nix            the app (whole file below); inputs: nixpkgs only
 A  evidence/flake.lock
 A  evidence/drive.sh             flags, composition, teardown trap
 A  evidence/lib/drive.ts         readiness → { page, shot, serverLog } → section import
 A  evidence/lib/store.sh         store copy, cwd rewrite, adapter env (#417's plumbing)
 A  evidence/lib/video.ts         filming, when a section says `export const record = true`
 A  evidence/sections/example.ts  the starting point a lane copies
 A  evidence/host/serve.sh        ★ the ONE olai-specific file: how THIS app boots
 A  evidence/host/fixtures/small/ tiny default vault
 A  evidence/README.md            contract + upstreaming note
 M  HACKING.md                    one line: "evidence: see evidence/README.md"
```

The `★` is the extraction seam: `lib/` and `drive.sh` know nothing about olai — they run "a web app" defined by `host/serve.sh` (boot command, readiness URL, log path). Upstreaming = `git filter-repo` the folder into its own repo, delete `host/`, and each consumer writes its own `host/`. No imports cross the folder boundary in either direction; helpers were absorbed, not imported.

```
evidence/
├── flake.nix
├── flake.lock
├── drive.sh
├── lib/
│   ├── drive.ts
│   ├── store.sh
│   └── video.ts
├── sections/
│   └── example.ts
├── host/                ← the only part that stays behind at upstreaming
│   ├── serve.sh
│   └── fixtures/small/
│       ├── house.olai
│       ├── roadmap.olai
│       └── notes.md
└── README.md
```

## evidence/flake.nix — whole

```nix
{
  description = "drive: photograph a web app doing the thing, against real data";

  inputs.nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";

  outputs = { self, nixpkgs }:
    let
      forAllSystems = f: nixpkgs.lib.genAttrs
        [ "x86_64-linux" "aarch64-darwin" ]
        (system: f nixpkgs.legacyPackages.${system});
    in {
      apps = forAllSystems (pkgs: {
        default = {
          type = "app";
          program = nixpkgs.lib.getExe (pkgs.writeShellApplication {
            name = "drive";
            runtimeInputs = [ pkgs.bun pkgs.playwright-driver.browsers pkgs.ffmpeg ];
            text = ''
              export PLAYWRIGHT_BROWSERS_PATH=${pkgs.playwright-driver.browsers}
              exec bash ${./drive.sh} "$@"
            '';
          });
        };
      });
    };
}
```

`host/serve.sh` is read at runtime by `drive.sh` (not baked into the flake), so a consumer repo can change how its app boots without touching the app derivation:

```bash
# evidence/host/serve.sh — the olai adapter, whole
serve_build()  { (cd "$REPO_ROOT" && just build-client); }
serve_start()  { (cd "$REPO_ROOT" && bun run olai web --port "$1" --dir "$2" ${STORE_HOME:+--acp}) & echo $!; }
serve_ready()  { curl -sf "http://127.0.0.1:$1/health"; }
serve_logs()   { echo "$XDG_STATE_HOME/olai/server.log"; }
```

## The section contract

```ts
// my-evidence.ts — dismiss × on a quiet subagent, resume it, see it return
import type { Drive } from "./evidence/lib/drive.ts"

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

```
$ nix run ./evidence -- --store ~/.claude ./my-evidence.ts
drive: client → built (cached)
drive: store  → /tmp/drive-h4Xk/home/.claude    (33 sessions, cwd → /tmp/drive-h4Xk/vault)
drive: server → http://127.0.0.1:43117           (up 2.1s, hydrated)
drive: shot   → shots/my-evidence.strip-before-dismiss.png
drive: shot   → shots/my-evidence.strip-after-dismiss.png
drive: shot   → shots/my-evidence.strip-resumed-returns.png
drive: clean  (3 shots, 11.4s)
```

```
drive: FAIL in section at shot "strip-resumed-returns" — TimeoutError: waitFor …
drive: server log tail ↓
  [strip] membership: pr-author dismissed=true    ← the bug, visible
drive: kept 2 shots in ./shots; exit 1
```

Briefs shrink to: *"Evidence via `nix run ./evidence`; section inline in the PR body beside its shots."* The sectionless form is the human's one-command "run olai against my real data."

---

Tier: **ordinary** → grok authors, claude-opus reviews.
