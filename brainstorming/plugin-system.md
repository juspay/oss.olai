# A plugin system for olai

*2026-08-30, with the human. Rulings: kolu and odu become plugins — everything integration-shaped (events, live properties, MCP, file conventions, docs). Compiled-in is fine; the point is separation of concerns. Core must stay polymorphic — no plugin names in general code. Enabled plugins are a CLI/nix-only setting (like git policy, #434); prefs shows them read-only.*

## Today

Kolu was extracted once (`kolu-client`, `kolu-ui`) but general packages still name it in ~55 files:

```
packages/
  chat/       kolu.ts (341 lines: probe + failure prose)
              agent.ts        mcpServersOf(tools, kolu: Kolu.Server | null)   ← one tenant, by name
  server/     runtime.ts      kolu: {...} | null                              ← a named slot
              koluConfig.ts   the Kolu.olai file convention
  surface/    index.ts        ...koluMembers spread ×4 into the wire spec
  web/        props/blocks.ts registerBlock() is generic, but imports TERMINAL_KEY
              props/PropsDrawer.tsx   registerBlock(TERMINAL_KEY, TerminalBlock)  ← by name
              client/padi/    header pill, events drawer, wrench → mounted by name in App.tsx
  kolu-client/  ✅ extracted (depends on no olai package; KoluDeps = injected vault walks)
  kolu-ui/      ✅ extracted
  odu-client/   ✅ extracted (#433)
```

`TERMINAL_KEY` is spelled at 7 sites across 4 packages. That grep is the problem statement.

## Target

```
packages/
  plugins/          NEW — the interface + registries; the ONLY place core meets plugins
  plugin-kolu/      manifest + everything above that says "kolu" (wraps kolu-client/kolu-ui)
  plugin-odu/       manifest + odu-ci dressing + run events (wraps odu-client)
  chat/ server/ surface/ web/ ...   zero plugin names
```

## The interface

```ts
// packages/plugin-kolu/src/index.ts
export const plugin: OlaiPlugin = {
  name: "kolu",                                   // prefs row, docs slug
  // server
  probe,                                          // find the tool; absence = a state, not an error
  wireMembers,                                    // cells/collections/streams/procedures
  runtimeHalf,                                    // subscription machinery, deps injected (KoluDeps, generalized)
  mcpServer,                                      // handed to chat sessions when probe says yes
  ownedFile: { basename: "kolu.olai", read: watchConfigIn },
  // client
  dressings: [{ key: "terminal", block: TerminalBlock }],
  chrome: { header: PadiPill, drawer: EventsFeed },
  // tests
  testDrivers,                                    // fake-padi, @kolu / @padi: tags
}
```

```ts
// packages/plugins/src/registry.ts — assembled ONCE, at build
import { plugin as kolu } from "@olai/plugin-kolu"
import { plugin as odu } from "@olai/plugin-odu"
export const PLUGINS = [kolu, odu]
```

## Core, before → after

```ts
// chat: MCP handing
mcpServersOf(tools, kolu: Kolu.Server | null)          // before
mcpServersOf(tools, enabled(PLUGINS).map(p => p.mcpServer))  // after

// server: the runtime slot
RuntimeWiring = { ..., kolu: KoluHalf | null }         // before
RuntimeWiring = { ..., halves: RuntimeHalf[] }         // after

// web: the one registration
registerBlock(TERMINAL_KEY, TerminalBlock)             // before
for (const p of enabled(PLUGINS)) p.dressings.forEach(register)  // after

// server: file conventions
koluFileIn(paths)                                      // before
ownedFileIn(p.ownedFile, paths)  per enabled plugin    // after
```

## Enable/disable

CLI/nix only, exactly the git-policy shape (#434): a server flag / home-manager option, no settings file, no browser toggle. Prefs draws the plugin rows read-only — enabled or disabled, with where to change it.

```
olai serve --plugins kolu,odu        (or the nix module option; default: all built-in)
              │
   disabled ──┴──> exactly the no-kolu machine:
                   probe never runs · wire members answer absent · chrome unmounted
                   dressings unregistered · Kolu.olai drawn as an ordinary outline
```

Cheap because `runtime.ts` already runs with `kolu: null` and every surface already treats absence as ordinary. Disabled = a state that already exists.

## The fence, generalized

`scripts/check-kolu-deps.sh` (a kolu-name allowlist) becomes two lints in CI:

1. only `packages/plugins/` may import `plugin-*`
2. no general package spells a plugin name or key (the 7 `TERMINAL_KEY` sites are the canary)

## Honest limits of compiled-in

- The wire spec is compile-time: disabled members exist but answer absent; a third-party plugin needs a rebuild. Fine under the ruling.
- Chat's failure prose ("kolu is missing from this conversation: …") is plugin-owned whole sentences — the interface must carry sentences, not templates.
- "key-selects, type-licences" is unfinished: no `terminal` kind exists in PropType yet. Plugins declaring a property KIND is the end state; key-matching is the interim.

## Phases

1. **Interface + registries** — `OlaiPlugin`, the four mechanical seams (dressings, wire spread, runtime half, MCP list) go registry-driven; kolu still the one tenant; the lint lands.
2. **Name sweep** — chat/kolu.ts, koluConfig.ts, padi/ chrome, testids move into `plugin-kolu`; general packages hit zero plugin names. The big one.
3. **plugin-odu** — odu-client + odu-ci dressing + run events into the feed (absorbs odu-in-olai phase 3; phase 4 lives here or dies).
4. **Prefs** — the read-only plugin rows + the CLI/nix flag + disabled-equals-absent tests.

## Open questions

1. Does the chat agent roster (claude/opencode/pi — a second de-facto registry) eventually ride this interface?
2. `@kolu/surface*` (the app framework) stays out of scope — it is the house, not an appliance. Confirm.
3. Does `docs/kolu.md` move into the plugin's directory?
