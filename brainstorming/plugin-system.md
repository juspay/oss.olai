# A plugin system for olai — kolu and odu become plugins

*Opened 2026-08-30 with the human. Rulings so far: the plugin boundary is TOTAL (events, live properties, MCP handing, file conventions, docs — everything integration-shaped); COMPILED-IN is fine — the point is separation of concerns, a real interface, and a low bar for the next plugin — but core must not entangle with non-polymorphic plugin code (if compiled interfaces cannot keep core free of plugin names, dynamic loading is back on the table); the user sees plugins in PREFS and can enable/disable them.*

## Why (the human's framing)

The main benefit is separation of concerns: olai's general code should carry no kolu or odu names. A proper plugin interface makes the next integration cheap, and makes "what does this plugin touch" one readable thing instead of a grep. Compiled-in is acceptable because the boundary is the value, not the loading mechanism — but the boundary must be real: polymorphic receptacles in core, appliances in plugins.

## Ground truth (repo survey, 2026-08-30, origin/master = df6da10e)

The extraction has happened ONCE already and stalled at the package line. The sixth lowy-electricity sitting moved kolu implementation into `@olai/kolu-client` (26 files, depends on no olai package) and `@olai/kolu-ui` (22 files), fenced by `scripts/check-kolu-deps.sh` — a hardcoded allowlist asserting only those two packages may import kolu's product-tier packages. odu just repeated the first step: `packages/odu-client` (the thin client, #433's pin) plus the `odu-ci` dressing.

But **55 non-test files in general packages still carry genuine kolu-specific code** (133 mention kolu/padi at all; the rest is the unrelated `@kolu/surface*` framework tier or comments). The big ones, each a receptacle in disguise:

- `packages/chat/src/kolu.ts` — 341 lines of probe judgement: the 3-arm detection, the EXPECTED sentence, five English failure explanations, `PADI_SOCKET` awareness.
- `packages/chat/src/agent.ts:2315` — `mcpServersOf(tools, kolu: Kolu.Server | null)`: the MCP handing takes exactly one optional integration, by name, in its type.
- `packages/server/src/runtime.ts` — the runtime's `kolu: {...} | null` half: a NAMED optional slot, spread into cells/collections/streams/procedures handlers at four sites.
- `packages/server/src/koluConfig.ts` — the `Kolu.olai` file convention (basename resolution + knob grammar + mute reading), spelled in the server while format spells the other two conventions.
- `packages/surface/src/index.ts` — `koluMembers` spread into the wire spec at four places (cells `kolu`/`pulse`/`mutes`, collections `fleet`/`events`, stream, procedure) + a re-export tail.
- `packages/web/src/client/props/blocks.ts` — the dressing registry (`registerBlock`) is generic, but the file imports and re-exports `TERMINAL_KEY`; the app's one registration happens in `PropsDrawer.tsx` by name. `TERMINAL_KEY` is spelled at seven call sites across four packages.
- `packages/web/src/client/padi/` — the header pill, the events drawer, the mutes foot, the wrench onto `Kolu.olai`; mounted by name in `AppHeader.tsx` and `App.tsx`.
- Plus: kolu-flavored chat notices, testids in both packages, `docs/kolu.md` and chat.md's kolu section, the fake-padi test harness with `@kolu`/`@padi:` cucumber tags, nix seeds, env vars.

Two facts worth their weight:

1. **The cleanest socket already exists**: `KoluDeps` (`kolu-client/src/index.ts`) — the vault walks are *injected*, so kolu-client never learns olai's node type. The plugin interface generalizes this, not invents it.
2. **Absence is already ordinary**: a machine without kolu draws nothing and errors nothing, and `runtime.ts` already supports `kolu: null` with every wire member answering absent. That grace is precisely what makes a prefs *disable* cheap: disabled = the absent machine, a state every surface already handles.

The surveyor also found a SECOND de-facto plugin registry nobody asked about: the chat agent roster (`chat/src/agents/{claude,opencode,pi}.ts`) — three named agents behind one interface, with their own env vars. Not this doc's scope, but the plugin system should not be designed in a way that couldn't one day take it.

## The shape

**One manifest per plugin; core builds every registry from the manifests; general code never names a plugin.**

```ts
// packages/plugin-kolu — the manifest is the whole public story
export const plugin: OlaiPlugin = {
  name: "kolu",                          // prefs row, docs slug, testid prefix
  // server half
  probe,                                 // detect the tool; absence is a state, not an error
  wireMembers,                           // cells/collections/streams/procedures into the spec
  runtimeHalf,                           // the deps-injected subscription machinery (KoluDeps generalized)
  mcpServer,                             // what to hand chat sessions when the probe says yes
  ownedFile: { basename: "kolu.olai", reads: watchConfigIn },  // the convention + its grammar
  // client half
  dressings: [{ key: "terminal", block: TerminalBlock }],      // key today, declared KIND when licensing lands
  chrome: { header: PadiPill, drawer: EventsFeed },            // mounted by the registry, not by App.tsx
  docs: koluDocPage,
  // test half
  testDrivers,                           // fake-padi, tags — the harness asks plugins, not the reverse
}
```

- **Registries replace name-sites.** `App.tsx` mounts `plugins.chrome`, `PropsDrawer` registers `plugins.dressings`, `mcpServersOf(tools, plugins.mcpServers)` takes a list, the runtime takes `halves: RuntimeHalf[]`, the surface spec spreads `plugins.wireMembers`, the file-convention resolver walks `plugins.ownedFiles`. Each of these is a one-shape change at an existing seam — the survey shows every one already has exactly one tenant plus generic machinery around it.
- **The polymorphism bar, testably.** The fence script generalizes from "only kolu packages import kolu deps" to two lints: (a) no general package imports a `plugin-*` package except the one registry assembly point; (b) no general package spells a plugin's name/keys (the seven `TERMINAL_KEY` sites become the canary). The bar the human set — no entanglement with non-polymorphic plugin code — becomes CI-enforced, not reviewed-for.
- **Prefs enable/disable.** A plugins row in preferences, one toggle per manifest. Disabled ⇒ the probe never runs, the wire members answer the same absent state a kolu-less machine answers, chrome unmounted, dressings unregistered, the owned file ignored (drawn as an ordinary outline). Because absence is already ordinary everywhere, disable buys honesty for free; the work is plumbing the toggle to the registries and deciding its scope (per-browser preference like Done:Visible, or server-side state — probes and MCP handing are server acts, so a purely client toggle can only hide, not stop; likely: server-side enablement, client draws what the server enabled).

## What compiled-in cannot do (the honest line)

- The wire SPEC is compile-time: a disabled plugin's members still exist in the spec (answering absent), and a third-party plugin can't add members without a rebuild. Acceptable under the compiled ruling; worth saying out loud.
- Chat's five failure sentences and the EXPECTED prose are plugin-owned strings behind a generic "integration status" surface — the registry can carry them, but the *quality* of those sentences is why the kolu integration feels good; the interface must let a plugin say whole sentences, never fill templates.
- The dressing licence ("key-selects, type-licences") is still future work: `PropType` has no `terminal` or `duration` kind today. The plugin system forces that question — a plugin declaring a property KIND plus its dressing is the clean end-state; key-matching is the interim.

## Phases (sketch, sized by the survey)

1. **The interface + registries** — define `OlaiPlugin`, build the registries at the assembly point, move the FOUR mechanical registrations (dressings, wire spread, runtime half, MCP list) behind it. kolu still the one tenant; no behavior change; the generalized lint lands.
2. **The name sweep** — chat/kolu.ts, koluConfig.ts, padi/ chrome, testids, notices move into `packages/plugin-kolu` (re-exporting kolu-client/kolu-ui internals); the ~55 general-package files go to zero plugin names. The biggest phase; it is the "separation of concerns" the human named.
3. **The odu plugin** — `packages/plugin-odu` wraps odu-client + the odu-ci dressing + run-event feed sources (absorbing the odu-in-olai family's phase 3; phase 4's launch/classification lives inside this plugin or dies — the MCP argument from chat, 2026-08-30).
4. **Prefs** — the toggle row, server-side enablement, the disabled-equals-absent tests.

## Open questions for the human

- Server-side enable/disable (a machine fact) vs per-browser (a preference)? The probe/MCP halves argue server-side; is a per-vault setting acceptable, and where does it live (the settings story just went CLI-only in #434)?
- Does the agent roster (claude/opencode/pi) eventually ride the same interface, or stay its own seam?
- Is `@kolu/surface*` (the app framework olai is built on) explicitly out of scope forever, or does the plugin story eventually want olai not to be a kolu-framework app? (This doc assumes: out of scope — it is the house, not an appliance.)
- Docs shipping: does `docs/kolu.md` stay a repo doc owned by the plugin directory, or become a page the plugin registers?
