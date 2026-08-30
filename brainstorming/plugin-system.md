# A plugin system for olai

*2026-08-30, with the human. All questions resolved — this doc is ready to become a lane brief.*

**Rulings:** kolu and odu become plugins — everything integration-shaped (events, live properties, MCP, file conventions, docs). Compiled-in; core stays polymorphic — no plugin names in general code, CI-enforced. Enabled plugins are a CLI/nix-only setting (like git policy, #434); prefs shows the rows read-only. Failure prose is the plugin's — whole sentences, never core templates. Property types are solved NOW (kinds as data through the format), not name-matching. The interface is shaped so chat agents COULD ride it one day (roster untouched now). `@kolu/surface` (the framework) stays — it is the house, not an appliance. A plugin's docs live in its own directory; the docs index assembles pages from plugins. The whole thing lands as ONE PR.

## Today

Kolu was extracted once (`kolu-client`, `kolu-ui`) but general packages still name it in ~55 files:

```
packages/
  chat/       kolu.ts (341 lines: probe + failure prose)
              agent.ts        mcpServersOf(tools, kolu: Kolu.Server | null)   ← one tenant, by name
  server/     runtime.ts      kolu: {...} | null                              ← a named slot
              koluConfig.ts   the Kolu.olai file convention
  surface/    index.ts        ...koluMembers spread ×4 into the wire spec     ← kolu words in the core API
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
    docs.md         ← the plugin's user docs live HERE (ruled); the docs index assembles them
  plugin-odu/       manifest + odu-ci dressing + run events (wraps odu-client)
  chat/ server/ surface/ web/ ...   zero plugin names
```

## The API is generic — no plugin words in it

Core's API does NOT carry "list the terminals". It carries ONE generic mount, and each enabled plugin's members hang under their own namespace:

```
before   surface spec:  cells kolu/pulse/mutes, collections fleet/events, ...   ← kolu words, compile-time
after    surface spec:  plugins/<name>/<member>                                 ← one generic door

         plugins/kolu/fleet      plugins/kolu/mutes      (types live in plugin-kolu)
         plugins/odu/run         plugins/odu/events      (types live in plugin-odu)
```

The plugin's own package declares its member types; its own client half (the dressings, the drawer) imports them from itself. Core routes the namespace and knows nothing else. A DISABLED plugin's namespace is simply not mounted — the same absent state a machine without the tool already shows, answered uniformly at the one generic door.

## The interface

```ts
// packages/plugin-kolu/src/index.ts
export const plugin: OlaiPlugin = {
  name: "kolu",                                   // namespace, prefs row, docs slug
  // server
  probe,                                          // find the tool; absence = a state, not an error
  members,                                        // this plugin's cells/collections/streams/procedures,
                                                  //   mounted under plugins/kolu/ — typed HERE, not in core
  runtimeHalf,                                    // subscription machinery, deps injected (KoluDeps, generalized)
  mcpServer,                                      // handed to chat sessions when probe says yes
  ownedFile: { basename: "kolu.olai", read: watchConfigIn },
  failures,                                       // WHOLE sentences per failure case — core displays, never composes
  // format
  propKinds: [{ kind: "terminal", admits: isTerminalId }],
  // client
  dressings: [{ kind: "terminal", block: TerminalBlock }],   // licensed by declared KIND, not key name
  chrome: { header: PadiPill, drawer: EventsFeed },
  docs,                                           // the docs.md page the index assembles
  // tests
  testDrivers,                                    // fake-padi, @kolu / @padi: tags
}
```

```ts
// packages/plugins/src/registry.ts — assembled ONCE, at build; filtered by the CLI flag at boot
import { plugin as kolu } from "@olai/plugin-kolu"
import { plugin as odu } from "@olai/plugin-odu"
export const PLUGINS = [kolu, odu]
```

The interface is deliberately roomy enough that a chat AGENT (claude/opencode/pi — today a second hardcoded roster in `chat/src/agents/`) could one day be a plugin: probe + failure sentences + per-conversation attach is the same shape. Ruled: design for it, migrate later; the roster is untouched in this PR.

## Property types now

Today a prop named `terminal` gets the terminal door — name-matching. Ruled: solve types in this PR. A plugin contributes property KINDS; the vault declares the kind in `_olai/Properties.olai`; the face follows the KIND, whatever the prop is named.

Kept polymorphic: `@olai/format` does not import plugins — its kind vocabulary becomes a parameter. The server assembles the kind table from enabled manifests and hands it to the validator as data (the same move `KoluDeps` makes with vault walks). A kind whose plugin is disabled validates as plain text — the value is still a name; nothing breaks — and wears no face.

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
for (const p of enabled(PLUGINS)) p.dressings.forEach(register)  // after — keyed by declared kind

// server: file conventions
koluFileIn(paths)                                      // before
ownedFileIn(p.ownedFile, paths)  per enabled plugin    // after
```

## Enable/disable

CLI/nix only, exactly the git-policy shape (#434): a server flag / home-manager option, no settings file, no browser toggle. Prefs draws the plugin rows read-only — enabled or disabled, with where to change it.

```
olai serve --plugins kolu,odu        (or the nix module option; default: all built-in)
              │
   disabled ──┴──> exactly the machine-without-the-tool:
                   probe never runs · plugins/<name>/ not mounted · chrome unmounted
                   dressings unregistered · its kinds validate as text · Kolu.olai an ordinary outline
```

Cheap because absence is already ordinary everywhere (`runtime.ts` runs with `kolu: null` today). Disabled = a state that already exists.

## The fence, generalized

`scripts/check-kolu-deps.sh` (a kolu-name allowlist) becomes two lints in CI:

1. only `packages/plugins/` may import `plugin-*`
2. no general package spells a plugin name or key (the 7 `TERMINAL_KEY` sites are the canary)

## What compiled-in still cannot do

One thing only: a third party adding a NEW plugin rebuilds olai. Accepted — the boundary is the value, not the loading.

## The PR (one, ruled)

Sequenced as commits, each green on its own:

1. the interface + the generic mount + registries (kolu still the one tenant; the lint lands)
2. property kinds as data through the format (the licence flips from key to kind)
3. the name sweep — chat/kolu.ts, koluConfig.ts, padi/ chrome, testids into `plugin-kolu`; general packages hit zero plugin names
4. `plugin-odu` — odu-client + odu-ci dressing + run events into the feed (absorbs odu-in-olai phase 3; phase 4 lives here or dies)
5. the CLI/nix flag + read-only prefs rows + plugin docs assembly + disabled-equals-absent tests
