# drive — photograph an app doing the thing

    nix run ./evidence

Reads ./evidence.ts (your section: a small playwright script — click this,
type that), starts the app, runs it in a headless browser, writes one
screenshot per named step into ./shots/, tears everything down.
No ./evidence.ts → prints the example to copy. No flags, no arguments.

App knowledge lives in one place: host/serve.sh, a plain script that starts
your app. The environment is the whole contract.

## Conventions

    ./evidence.ts    your section. The well-known filename, like Makefile.
    .drive/home      the app's HOME. Created empty; never wiped; seed by writing into it.
    .drive/data      what the app serves. Absent: host/fixtures/default.
    .drive/app.log   the app's stdout+stderr, captured by drive.
    ./shots/         one png per shot() call.

    ready  = the app's port answers 200
    fresh  = rm -rf .drive
    video  = `export const record = true` in the section (mp4 instead of stills)

drive knows no app and no formats. Seeding .drive/* with meaningful state is
the caller's business.

## Exit status

    0   section completed, shots written
    1   section threw. app.log tail printed; shots so far and .drive/ kept.
    2   app failed to boot. app.log printed whole.

Always: every process drive started is dead on exit.

## Section

Default-export one async function; `page` arrives past readiness.

    // evidence.ts
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

Sections are throwaway: never committed; pasted into the PR body (a
`<details>` block) beside their shots.

## Run

    $ nix run ./evidence
    drive: home  → .drive/home    data → .drive/data (seeded)
    drive: app   → up 2.1s, ready
    drive: shot  → shots/before-dismiss.png
    drive: shot  → shots/after-dismiss.png
    drive: shot  → shots/resumed-returns.png
    drive: clean (3 shots, 11.4s)

## Layout

    evidence/
    ├── flake.nix            the runner; inputs: nixpkgs only
    ├── flake.lock
    ├── drive.sh             composition, teardown trap
    ├── lib/
    │   ├── drive.ts         readiness → { page, shot } → section import
    │   └── video.ts         record = true
    ├── example.evidence.ts  what gets printed when ./evidence.ts is missing
    ├── host/                ← the ONLY app-aware part
    │   ├── serve.sh
    │   └── fixtures/default/
    └── README.md

Nothing outside the folder except one pointer line in HACKING.md.
No imports cross the folder boundary.

## host/serve.sh — the adapter, whole

Starts the app in the foreground. drive supplies the environment and owns
the process, its output, and its death.

    #!/usr/bin/env bash
    # env: PORT (bind here), HOME (already .drive/home), DATA (serve this)
    set -euo pipefail
    cd "$REPO_ROOT"
    just build-client
    exec bun run olai web --port "$PORT" --dir "$DATA"

Port another app: copy evidence/, rewrite this script.
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
