# drive — photograph an app doing the thing

    nix run ./evidence -- <section.ts>

Starts the app, opens a headless browser on it, runs the section (a small
playwright script: click this, type that), writes one screenshot per named
step into ./shots/, tears everything down.

The tool is app-neutral. All app knowledge lives in host/serve.sh, a
four-function adapter. A web app is a server process; drive starts it and
stops it, nothing more.

## Files

    .drive/home     the app's HOME for the run. Created empty; never wiped.
    .drive/data     what the app serves. Absent: host/fixtures/default.
    ./shots/        output. One png per shot() call, section-prefixed.

Seed state by writing into .drive/* before running. drive knows no formats.
`rm -rf .drive` for a fresh run.

## Exit status

    0   section completed, shots written
    1   section threw. App log tail printed; shots so far and .drive/ kept.
    2   app failed to boot. Full log printed.

Always: every process drive started is dead on exit.

## Section

Default-export one async function. `page` arrives past readiness.
`export const record = true` films the run (mp4) instead of stills.

    // my-evidence.ts
    import type { Drive } from "./evidence/lib/drive.ts"

    export default async ({ page, shot }: Drive) => {
      await page.getByRole("button", { name: "agent" }).click()
      await page.getByText("pr-author").waitFor()
      await shot("before-dismiss")

      await page.getByLabel("dismiss pr-author").click()
      await shot("after-dismiss")

      await page.getByPlaceholder("ask the agent…").fill("@pr-author also fix the docs")
      await page.keyboard.press("Enter")
      await page.getByText("pr-author").waitFor({ timeout: 15_000 })
      await shot("resumed-returns")
    }

Sections are throwaway: not committed; pasted into the PR body (a
`<details>` block) beside their shots.

## Run

    $ nix run ./evidence -- ./my-evidence.ts
    drive: build  → ok (cached)
    drive: home   → .drive/home    data → .drive/data (seeded)
    drive: app    → up 2.1s, hydrated
    drive: shot   → shots/my-evidence.before-dismiss.png
    drive: shot   → shots/my-evidence.after-dismiss.png
    drive: shot   → shots/my-evidence.resumed-returns.png
    drive: clean  (3 shots, 11.4s)

## Layout

    evidence/
    ├── flake.nix            app runner; inputs: nixpkgs only
    ├── flake.lock
    ├── drive.sh             composition, teardown trap
    ├── lib/
    │   ├── drive.ts         readiness → { page, shot, appLog } → section import
    │   └── video.ts         record = true
    ├── sections/example.ts  copy me
    ├── host/                ← the ONLY app-aware part
    │   ├── serve.sh
    │   └── fixtures/default/
    └── README.md

Nothing outside the folder except one pointer line in HACKING.md.
No imports cross the folder boundary.

## host/serve.sh — the adapter, whole

    host_build()  { (cd "$REPO_ROOT" && just build-client); }
    host_start()  { (cd "$REPO_ROOT" && HOME="$DRIVE_HOME" bun run olai web --port "$1" --dir "$DRIVE_DATA") & echo $!; }
    host_ready()  { curl -sf "http://127.0.0.1:$1/health"; }
    host_logs()   { echo "$DRIVE_HOME/.local/state/olai/server.log"; }

Port another app: copy evidence/, rewrite these four functions.
Upstream one day: the folder minus host/ becomes its own repo.

## flake.nix

    {
      description = "drive: photograph a web app doing the thing";
      inputs.nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
      outputs = { self, nixpkgs }:
        let forAllSystems = f: nixpkgs.lib.genAttrs
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
