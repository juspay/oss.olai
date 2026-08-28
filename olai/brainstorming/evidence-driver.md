# `evidence/` — photograph an app doing the thing

*Brainstorming, 2026-08-28. Ruled: `nix run`; TypeScript sections; sections throwaway (inlined in the PR body beside their shots); everything encapsulated in `evidence/`, app-neutral, one day its own repo.*

**What it is, in one breath:** screenshot an app doing X, in one command. It starts the app in the background (isolated: `HOME` and data are scratch dirs in the worktree), opens a headless browser on it, runs your ten-line click-this-type-that script saving a picture at each named step, and shuts everything down. You get a `shots/` folder. The tool has no server of its own — a web app *is* a server process, so "the app running" means that process running; the tool starts it and stops it.

**The tool knows no app.** Everything app-specific — how to build, boot, and detect readiness, what default data to serve — lives in one adapter: `host/`. This repo's `host/` boots olai; another repo copies `evidence/` and writes its own four functions. The doc mentions no app outside that section.

**Why build it:** lanes proving their PRs rebuilt exactly this scaffolding three separate times on 2026-08-28, each copy dying with its worktree.

## Interface

```bash
nix run ./evidence -- <section.ts>
```

No flags. Everything else is a convention directory, seeded by writing into it:

```
.drive/home     the app's HOME for the run    (created empty if absent; never wiped)
.drive/data     what the app serves           (absent → host/fixtures/default)
./shots/        where the pictures land       (one per shot() call, section-prefixed)

rm -rf .drive   → a fresh run

exit 0  section done          exit 2  app failed to boot (full log printed)
exit 1  section threw (log tail printed; shots-so-far and .drive/ kept)
```

Want the app to see particular state — sessions, config, corpora? Put the files in `.drive/home` / `.drive/data` before running. How to fabricate that state is the caller's business; the tool never learns any format.

## The PR, as a diff tree — one `M`, everything else inside the folder

```
 A  evidence/flake.nix             the app runner (whole file below); inputs: nixpkgs only
 A  evidence/flake.lock
 A  evidence/drive.sh              composition + teardown trap
 A  evidence/lib/drive.ts          readiness → { page, shot, appLog } → section import
 A  evidence/lib/video.ts          filming, when a section says `export const record = true`
 A  evidence/sections/example.ts   the starting point a lane copies
 A  evidence/host/serve.sh         ★ the app adapter (this repo's: olai)
 A  evidence/host/fixtures/default/  the data served when .drive/data is absent
 A  evidence/README.md             contract + upstreaming note
 M  HACKING.md                     one line: "evidence: see evidence/README.md"
```

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
├── host/                ← the ONLY app-aware part; stays behind at upstreaming
│   ├── serve.sh
│   └── fixtures/default/
└── README.md
```

Upstreaming = `git filter-repo` the folder into its own repo minus `host/`; each consumer writes its own `host/`. No imports cross the folder boundary in either direction.

## evidence/flake.nix — whole

```nix
{
  description = "drive: photograph a web app doing the thing";

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

## host/serve.sh — the whole adapter contract

Four functions, read at runtime by `drive.sh`. This is the single place the app's name appears:

```bash
# evidence/host/serve.sh — this repo's adapter (olai)
host_build()  { (cd "$REPO_ROOT" && just build-client); }
host_start()  { (cd "$REPO_ROOT" && HOME="$DRIVE_HOME" bun run olai web --port "$1" --dir "$DRIVE_DATA") & echo $!; }
host_ready()  { curl -sf "http://127.0.0.1:$1/health"; }
host_logs()   { echo "$DRIVE_HOME/.local/state/olai/server.log"; }
```

## The section contract

Default-export one async function. `Drive` hands it a `page` already past readiness, `shot(name)`, and `appLog()`. A real worked example (a section is inherently a script *against the host app* — this one is the strip-resumed-agent lane's):

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
$ nix run ./evidence -- ./my-evidence.ts
drive: build  → ok (cached)
drive: home   → .drive/home    data → .drive/data (seeded)
drive: app    → up in 2.1s, hydrated
drive: shot   → shots/my-evidence.strip-before-dismiss.png
drive: shot   → shots/my-evidence.strip-after-dismiss.png
drive: shot   → shots/my-evidence.strip-resumed-returns.png
drive: clean  (3 shots, 11.4s)
```

```
drive: FAIL in section at shot "strip-resumed-returns" — TimeoutError: waitFor …
drive: app log tail ↓
  [strip] membership: pr-author dismissed=true    ← the bug, visible
drive: kept 2 shots in ./shots; exit 1
```

Briefs shrink to: *"Evidence via `nix run ./evidence`; section inline in the PR body beside its shots."*

---

Tier: **ordinary** → grok authors, claude-opus reviews.
