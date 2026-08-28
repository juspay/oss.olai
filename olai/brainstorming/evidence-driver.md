# The evidence driver — `nix run ./evidence`

*Brainstorming, 2026-08-28. Ruled: `nix run`; TypeScript sections; sections throwaway (inlined in the PR body beside their shots); **everything lives in `evidence/` with as little as possible leaking to the repo root — the folder is a future standalone repo** (the vault extraction already proved the wholesale-with-history move).*

**What it is, in one breath:** screenshot the app doing X, in one command. It starts olai in the background (the real app, serving a test folder, `HOME` pointed at a scratch dir so nothing real is touched), opens a headless browser on it, runs your ten-line click-this-type-that script saving a picture at each named step, and shuts everything down. You get a `shots/` folder. Note the server is not something the tool adds — the server IS olai (the browser loads the page from, and reads every outline through, the `olai web` process); the tool merely starts it and stops it, the way you couldn't photograph Gmail without Gmail running.

**Why build it:** lanes proving their PRs rebuilt exactly this scaffolding three separate times on 2026-08-28 (#416, #417, strip-resumed-agent), each copy dying with its worktree. `packages/tests` has adjacent machinery (`evidence.sh`, `serve.sh`, `paneVideo.ts`) but this PR deliberately does NOT refactor it onto the new folder — that coupling would chain the folder to this repo; it can migrate the day the folder is upstreamed and comes back as an input.

## Invocation

```bash
# Own flake — nothing registered on the root flake. From any worktree:

nix run ./evidence -- ./my-evidence.ts    # run section, produce shots. That's the tool.

# A lane that wants real Claude sessions seeds the run's HOME first —
# the DRIVER never learns what a session store is:
./evidence/host/claude-seed.sh ~/.claude .drive/home
nix run ./evidence -- ./my-evidence.ts

# "Just run the app against my data" is NOT this tool — the product already has it:
HOME=.drive/home olai web --dir path/to/vault
```

```
nix run ./evidence -- [--vault <dir>] <section.ts>

  section.ts       worktree-local section module (throwaway)
  --vault <dir>    .olai directory to serve   [default: evidence/host/fixtures/small]

The run's HOME is always `.drive/home` in the worktree: created if absent, never
wiped if present — seeding is putting files there before you run. Isolated from
the developer's real HOME, inspectable after a failure, `rm -rf .drive` for fresh.

The app is client-server, so the driver boots the server — but only through
host/serve.sh; ports, URLs and log paths are the host adapter's business and
never surface in this interface.

Not flags, not modes, on purpose:
  home   → always .drive/home (seed by writing into it)
  shots  → always ./shots (the section names each one)
  video  → the section's call: `export const record = true` films the run
  hold   → cut; standing the app up by hand is the product's own CLI, above

exit 0  section done          exit 2  boot failed (full server log printed)
exit 1  section threw (log tail printed; shots-so-far and .drive/ kept)
```

## The PR, as a diff tree — one `M`, everything else inside the folder

```
 A  evidence/flake.nix            the app (whole file below); inputs: nixpkgs only
 A  evidence/flake.lock
 A  evidence/drive.sh             flags, composition, teardown trap
 A  evidence/lib/drive.ts         readiness → { page, shot, serverLog } → section import
 A  evidence/host/claude-seed.sh  optional: copy a Claude store INTO .drive/home, rewrite
                                   transcript cwds to the served vault (#417's plumbing —
                                   host-side, because it knows Claude's formats)
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
│   └── video.ts
├── sections/
│   └── example.ts
├── host/                ← the only part that stays behind at upstreaming
│   ├── serve.sh
│   ├── claude-seed.sh
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
serve_start()  { (cd "$REPO_ROOT" && HOME="$DRIVE_HOME" bun run olai web --port "$1" --dir "$2") & echo $!; }
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
$ ./evidence/host/claude-seed.sh ~/.claude .drive/home
seed: 33 sessions → .drive/home/.claude (cwds → this worktree's vault)
$ nix run ./evidence -- ./my-evidence.ts
drive: client → built (cached)
drive: home   → .drive/home                      (33 sessions found)
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

Briefs shrink to: *"Evidence via `nix run ./evidence`; section inline in the PR body beside its shots."*

---

Tier: **ordinary** → grok authors, claude-opus reviews.
