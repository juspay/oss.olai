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

## The mount is already in @kolu/surface — use it, don't rebuild it

*(Grounded in kolu.dev/surface/ref-surface — "Composing siblings" and neighbors. Added after the first implementation attempt hand-flattened plugin keys into core tables: `"plugins:odu:ci": odu.handlers.cells.ci` in runtime.ts, a spelled resource/tool table in faces.ts, `olai.cells["plugins:odu:ci"]` in App.tsx — the exact entanglement the framework's own composition API exists to prevent.)*

What the framework already provides:

- **`composeSurfaceContracts({ <key>: surface })`** — merges standalone surfaces into one group by per-sibling tag **prefixing** (`surface/<key>/<member>/<verb>`), never a bare merge. The answer carries **`siblings[key]`**, each sibling's members at its tagged prefix — so handler binding and client dispatch are keyed BY SIBLING "without re-deriving tag rules". The sibling key IS the plugin namespace.
- **`implementSurface(surface, deps)`** — a plugin implements ITS OWN surface in its own package; the returned `SurfaceRuntime.handlers` come already keyed by full wire tag. Core splices runtimes; it never spells a member key.
- **`exposeFaces(surfaces, maps)`** — the sibling-bundle twin of `exposeFace`: each plugin ships its own `ExposeMap` (`"<member>": "resource"`, `"<ns>.<verb>": tool`) and core passes the bundle through. The MCP face classification lives in the plugin, not a core table.
- **`extendSurface(base, ext)`** — noted for completeness: it composes a parent-local runtime onto a RE-SERVED remote base (mirroring), with loud tag collisions and ordered teardown. Not this PR's tool — the plugins are local siblings, not remote mirrors — but it proves handler-record union by wire tag is a supported composition mode.
- Collisions **fail loud** at boot across the flat per-name namespace; reserved `system/*` members stay the framework's.

So the shape is:

```ts
// each plugin package: its own surface + runtime + expose map, self-contained
export const surface = defineSurface({ cells: { ci: {...} }, ... })       // plugin-odu/src/wire.ts
export const runtime = implementSurface(surface, deps)                    // handlers pre-keyed
export const faces: ExposeMap<typeof surface> = { ci: "resource" }

// core, once, generically — the ONLY meeting point:
const enabled = PLUGINS.filter(inFlag)
const composed = composeSurfaceContracts(fromEntries(enabled.map(p => [p.name, p.surface])))
const facesBundle = exposeFaces(composed, fromEntries(enabled.map(p => [p.name, p.faces])))
// handlers: union of enabled plugins' runtime.handlers — already tagged, spliced not spelled
```

**What might still warrant an upstream ask — to be confirmed by USING the API, not asserted:**

1. A **browser-client typed sibling handle** (`client.siblings[key]` with the plugin's own types) if the client half doesn't already mirror the server's sibling keying — the composition doc promises "client dispatch keyed by sibling", so possibly nothing is missing.
2. Boot-time sibling filtering (the `--plugins` flag) is plain data over `composeSurfaceContracts`'s record argument — expected to need nothing upstream.
3. If either turns out rough in practice, the ask goes to kolu with the paired-drishti rule honored — filed then, on the human's word, never assumed now.

## The interface

```ts
// packages/plugin-kolu/src/index.ts
export const plugin: OlaiPlugin = {
  name: "kolu",                                   // sibling key, prefs row, docs slug
  // server
  probe,                                          // find the tool; absence = a state, not an error
  surface,                                        // defineSurface(...) — typed HERE, not in core
  runtime,                                        // implementSurface(...) — handlers pre-keyed by wire tag
  faces,                                          // the plugin's own ExposeMap (resource/tool per member)
  mcpServer,                                      // handed to chat sessions when probe says yes
  ownedFile: { basename: "kolu.olai", read: watchConfigIn },
  failures,                                       // WHOLE sentences per failure case — core displays, never composes
  // format
  propKinds: [{ kind: "terminal", admits: isTerminalId }],
  // client
  dressings: [{ kind: "terminal", block: TerminalBlock }],   // licensed by declared KIND, not key name
  chrome: { header: PadiPill, drawer: EventsFeed },          // chrome owns its own cell access — core hands the client down
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

## One plugin per agent (the eighth sitting's cut, 2026-08-31)

One plugin PER AGENT, never one roster plugin — the three rows don't share a clock (Claude's adapter pin moved five times in a month; opencode's zero). Each agent's grenade lands in its own vault.

```
packages/plugin-claude/
  src/
    leg.ts        ← chat/src/agents/claude.ts moves here whole (the _meta bets, mcp__ tool
                    spelling, steering, the queue advertisement — adapter-release frequency)
    models.ts     ← the two Claude-only bets that today sit in a general package
                    (picker id "model"; alias tiers sonnet → claude-sonnet-5)
    adapter.ts    ← the PIN and its patches (acp/patches/ — the 0.66.0 patch debt travels HERE)
    prompt.ts     ← the system-prompt CHANNEL (_meta.systemPrompt); the TEXT stays core's
    mark.tsx      ← the SVG (browser half; the generic fallback mark stays in web)
  docs.md
packages/plugin-opencode/   ← PATH probe, tool spelling `bash:0`, no-interrupt fact
packages/plugin-pi/         ← pi-acp pin + the pi-mcp-servers bridge patch + the PATH pair
```

The server half grows ONE optional hook — beside `probe`, never overloading it. `probe()` answers with a tool to hand **to** a session; an agent **is** the session. Same three fields (`command`/`args`/`env`), different door; the resemblance is a trap.

```ts
// @olai/plugins/server — the one new hook
export interface PluginAgent {
  readonly id: string                              // the picker's key — DATA, not a union arm
  readonly leg: Leg                                // this agent's wire bets (@olai/chat's socket type)
  readonly at: (where: Where) => Adapter | null    // find it on THIS host; null = not installed, no fault
  readonly missing: NotHere | null                 // the install sentence (NoAgent.tsx's rows, one per plugin)
  readonly prompt: PromptChannel                   // HOW the standing prompt rides (_meta vs first turn)
}

export interface PluginServerHalf<R> extends PluginWire {
  // …probe, runtime, kinds — unchanged
  readonly agent?: PluginAgent                     // ← the hook. Most agent plugins carry ONLY this.
}
```

```ts
// packages/plugin-claude/src/server.ts — the whole manifest, most fields empty (as OlaiPlugin's header promised)
export const server = {
  name: "claude",
  agent: {
    id: "claude",
    leg: CLAUDE,
    at: (where) => adapterFrom(where.env[AGENT_ENV] ?? PINNED),   // the pin lives with its bets
    missing: null,                                                // shipped-in: absence is never a fault
    prompt: viaMeta,
  },
} satisfies PluginServerHalf<Revision>
```

Core, before → after:

```ts
// @olai/surface chat.ts
export const AGENTS = { claude, opencode, pi }     // before: a closed union — a 4th agent is a core PR
readonly agents: ReadonlyArray<AgentRow>           // after: data the server sent; the picker draws what arrived

// @olai/chat agents/roster.ts
const KINDS = [ {id: "claude", leg, at}, … ]       // before: the second hardcoded roster
rosterOf(agents, where)                            // after: the composition root hands the enabled list in
```

What stays core: `@olai/acp` whole (the protocol is the LANGUAGE, not an integration — a general package every plugin may import); `Leg` the interface (the socket every bet is safe to lose against); the panel/picker/queueing UX; the system-prompt TEXT (one olai-authored module, versioned with the binary — only the channel is per-agent).

Enable/disable is the existing story, nothing new: `--plugins=opencode,pi` serves a panel with no Claude row, no probe run, no mark — and the no-agent face's install sentences are each enabled plugin's `missing.why`, spent a second time. `OLAI_ACP_AGENT=""` (panel off) keeps its meaning; `--plugins` without any agent plugin says the same thing in the system's own grammar.

Refused at the design, so the lane cannot drift into them: no probe overload (above); no dummy `defineSurface({})` to satisfy the wire type — either the type's required-surface clause argues itself down for this population, or an agent plugin declares a member it genuinely owns (a roster row is a cell candidate — the lane's design round decides); `@olai/acp` never dissolves into the three plugins (the one place "ACP" is spelled correctly stays one place).

## The second doorbell's door (the wake PR, 2026-08-31)

The doorbell is plugin-kolu's feature whole — the watcher, the derivation, the meanings, the digest, the saved events, the strip's scope control. Core grows exactly ONE generic capability: deliver a machine-marked message into a conversation. Prototype: [second-doorbell.html](../projects/olai/brainstorming/second-doorbell.html).

```ts
// @olai/plugins/server — PluginServices grows one field, handed in at the composition root
// (the same move that hands a probe's MCP server to chat: the ROOT wires, no plugin imports chat)
export interface Deliveries {
  /** The conversations that opted into THIS plugin's wakes, each with its chosen filter file.
   *  Core stores the pair beside the conversation note; the plugin never sees a transcript. */
  readonly scopes: () => ReadonlyArray<{ readonly conversation: ConversationId; readonly file: string }>
  /** ONE machine-marked message into one conversation. Core owns the mechanics:
   *  agent idle → starts a turn; busy → queues AT THE AGENT behind whatever the human
   *  queued first (never the composer, never a keystroke); no session → saved, delivered
   *  as the next session's first message. The WORDS are the plugin's — whole sentences,
   *  the missing.why rule spent a third time. */
  readonly deliver: (to: ConversationId, body: string, opts?: {
    /** Messages sharing a key, while still undelivered, REPLACE each other — the plugin
     *  sends a fresh combined digest per event and core holds exactly one. Combining
     *  stays the plugin's authorship; holding stays core's mechanics. */
    readonly coalesce?: string
  }) => Promise<void>
}

export interface PluginServerHalf<R> {
  // …probe, runtime, kinds, agent? — unchanged
  /** The strip control's SENTENCE, when this plugin wakes conversations: core draws
   *  the row and the file picker (files are core vocabulary), the plugin says what
   *  the wake IS — subject first, filter second. */
  readonly wake?: { readonly control: string }   // "wake on terminal activity · terminals from"
}
```

```ts
// packages/plugin-kolu/src/wake.ts — the doorbell, whole, in the plugin
export const wake = (services: PluginServices<Revision>) =>
  watcher.on((event) => {
    for (const { conversation, file } of services.deliveries.scopes()) {
      const set = claimedTerminals(file)        // kind kolu-terminal on un-done nodes, mirrors resolved
      const meaning = meaningOf(event, set)     // WAKE · digest · drift · silence, from the board
      if (meaning === "silence") continue
      services.deliveries.deliver(conversation, sentenceOf(event, meaning),
        meaning === "digest" ? { coalesce: "kolu-digest" } : undefined)
    }
  })
```

What each side owns:

| | core (the door) | plugin-kolu (the doorbell) |
|---|---|---|
| the strip row + file picker | draws it, stores `(conversation, plugin, file)` | says the control's sentence |
| delivery mechanics | idle/busy/no-session, queue order, the machine mark | — |
| the words | never composes one | every sentence, every meaning |
| the set + the meanings | — | derives from the board via the licence consult |
| saved events | holds them per scope | summarizes them on delivery |

Refused drifts: no kolu words in core — the door speaks conversations and files only; delivery never touches the composer (the wire path a queued send already rides); `deliver` is write-only — a plugin never reads a conversation; kolu disabled = the strip row absent, the doorbell absent, the honest machine-without-the-tool state. The Monitor retires against this door, and `_olai/Kolu.olai` reduces to the watch knobs.

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
RuntimeWiring = { ..., halves: enabled plugins' runtimes, spliced }   // after

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
                   probe never runs · the sibling is not composed · chrome unmounted
                   dressings unregistered · its kinds validate as text · Kolu.olai an ordinary outline
```

Cheap because absence is already ordinary everywhere (`runtime.ts` runs with `kolu: null` today). Disabled = a state that already exists.

## The fence, generalized

`scripts/check-kolu-deps.sh` (a kolu-name allowlist) becomes two lints in CI — never a per-plugin script copy:

1. only `packages/plugins/` may import `plugin-*`
2. no general package spells a plugin name or key (the 7 `TERMINAL_KEY` sites are the canary; a spelled `plugins:<name>:` string in core fails the same lint)

## What compiled-in still cannot do

One thing only: a third party adding a NEW plugin rebuilds olai. Accepted — the boundary is the value, not the loading.

## The PR (one, ruled)

Sequenced as commits, each green on its own:

1. the interface + sibling composition via `composeSurfaceContracts`/`exposeFaces` + registries (kolu still the one tenant; the lint lands)
2. property kinds as data through the format (the licence flips from key to kind)
3. the name sweep — chat/kolu.ts, koluConfig.ts, padi/ chrome, testids into `plugin-kolu`; general packages hit zero plugin names
4. `plugin-odu` — odu-client + odu-ci dressing + run events into the feed (absorbs odu-in-olai phase 3; phase 4 lives here or dies)
5. the CLI/nix flag + read-only prefs rows + plugin docs assembly + disabled-equals-absent tests
