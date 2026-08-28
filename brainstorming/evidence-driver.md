# saatchi — photograph an app doing the thing

*(Tamil சாட்சி: witness, evidence. A standalone repo: github.com/juspay/saatchi.)*

    nix run github:juspay/saatchi

Reads evidence/evidence.ts (your section: a small playwright script — click
this, type that), starts the app, runs it in a headless browser, writes one
screenshot per named step into evidence/shots/, tears everything down.
No flags, no arguments, ever.

The consumer keeps exactly ONE well-known directory. saatchi knows no app:
the app is whatever evidence/serve.sh starts, and the environment is the
whole contract between them.

## The consumer's directory

    evidence/
    ├── serve.sh        tracked    the adapter: starts YOUR app (whole file below)
    ├── fixtures/       tracked    default data when data/ is absent
    ├── .gitignore      tracked    scaffolded: everything below this line
    ├── evidence.ts     throwaway  the current section
    ├── home/           throwaway  the app's HOME; seed by writing into it
    ├── data/           throwaway  what the app serves, when present
    ├── app.log         throwaway  the app's stdout+stderr, captured by saatchi
    └── shots/          throwaway  one png per shot() call

    ready  = the app's port answers 200
    fresh  = git clean -fx evidence/
    video  = `export const record = true` in the section (mp4 instead of stills)

## UX, end to end

First run in a repo — no evidence/ yet — scaffolds it and stops:

    $ nix run github:juspay/saatchi
    saatchi: no evidence/ here — scaffolded one:
      evidence/serve.sh      ← EDIT ME: start your app (env: PORT, HOME, DATA)
      evidence/evidence.ts   ← the example section; make it yours
      evidence/fixtures/     ← default data to serve
      evidence/.gitignore
    saatchi: edit serve.sh, then run me again.

Every run after:

    $ nix run github:juspay/saatchi
    saatchi: app  → up 2.1s, ready
    saatchi: shot → evidence/shots/before-dismiss.png
    saatchi: shot → evidence/shots/after-dismiss.png
    saatchi: shot → evidence/shots/resumed-returns.png
    saatchi: clean (3 shots, 11.4s)

Failure — loud and situated:

    saatchi: FAIL at shot "resumed-returns" — TimeoutError: waitFor …
    saatchi: app.log tail ↓
      [strip] membership: pr-author dismissed=true   ← the bug, visible
    saatchi: kept 2 shots; evidence/ left as-is; exit 1

    0  section done   1  section threw   2  app failed to boot (app.log whole)
    Always: every process saatchi started is dead on exit.

## evidence/serve.sh — the adapter, whole (olai's)

Starts the app in the foreground; saatchi owns the process, its output, its
death. `PORT` to bind, `HOME` already pointed at evidence/home, `DATA` the
directory to serve.

    #!/usr/bin/env bash
    set -euo pipefail
    cd "$(git rev-parse --show-toplevel)"
    just build-client
    exec bun run olai web --port "$PORT" --dir "$DATA"

## The section

Default-export one async function; `page` arrives past readiness.

    // evidence/evidence.ts
    import type { Saatchi } from "saatchi"

    export default async ({ page, shot }: Saatchi) => {
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

## The saatchi repo

    saatchi/
    ├── flake.nix        the app; inputs: nixpkgs only
    ├── flake.lock
    ├── saatchi.sh       composition, teardown trap
    ├── lib/
    │   ├── drive.ts     readiness → { page, shot } → section import
    │   └── video.ts     record = true
    ├── scaffold/        what the first run writes into a bare repo
    │   ├── serve.sh
    │   ├── evidence.ts
    │   └── gitignore
    └── README.md        this document, near verbatim

    {
      description = "saatchi: photograph an app doing the thing";
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
                name = "saatchi";
                runtimeInputs = [ pkgs.bun pkgs.playwright-driver.browsers pkgs.ffmpeg ];
                text = ''
                  export PLAYWRIGHT_BROWSERS_PATH=${pkgs.playwright-driver.browsers}
                  exec bash ${./saatchi.sh} "$@"
                '';
              });
            };
          });
        };
    }

## Bootstrapping order

saatchi is born in its own repo from day one — no extraction later. olai
consumes it by scaffolding `evidence/` and deleting its per-lane plumbing;
briefs shrink to: "Evidence via `nix run github:juspay/saatchi`; section
inline in the PR body beside its shots."
