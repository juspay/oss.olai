# Cordis for olai: from a plugin registry to a plugin runtime

*2026-09-02. A fresh study of DeepSeek Harness's plugin system and the framework under it (Cordis), against olai's `packages/plugin-api` at `bd40ea03`, and a proposal. Nothing olai ruled before is taken as given here; §7 re-tries each ruling on merit. Sources: the paper "A Programming Paradigm for Spatiotemporal Composability" (arXiv 2608.25512, Shi/Zhang/Cui, DeepSeek-AI and PKU), `deepseek-ai/deepseek-harness` (`docs/architecture.md`, the Cordis primer and tutorial, the generated event catalog, `packages/core/{tools,agent,agent-loop,scope,system-prompt}`, `packages/guard/timeout-policy`, `packages/preset/agent-presets`), and `cordiverse/cordis` v4 (`packages/{core,loader,hmr,include,group}`).*

---

## 1. The one-paragraph version

olai's plugin system is a **registry**. A plugin is a value; three `as const` arrays list the values; `--plugins` filters them once at boot; the composition root builds a `PluginServices` blob per plugin and pushes it in. It answers "which plugins are in this build" precisely, and nothing changes after the process starts. Cordis is a plugin **runtime**. A plugin is a function that installs *revertible effects* into a shared *context*, declares the services it needs (`inject`), and the runtime activates it when they exist, tears it down when one leaves, and re-applies it when it returns. Every registration carries its own undo. Everything else is a consequence: runtime enable and disable, hot reload, per-session and per-agent scoping, plugin-consumes-plugin, a declarative config tree with overlays, out-of-tree plugins without a rebuild, and plugins an agent writes and mounts while olai keeps serving.

The proposal: **make Cordis the runtime under olai, server and browser**, keep the judgements that survive re-examination (the vault's row beats the plugin's claim; the doorbell cannot read; failure prose is the plugin's; the browser receives answers, not rules), and overturn the ones that were limitations wearing a rule's clothes (compiled-in only; no plugin depends on a plugin; enable is CLI-only; the orchestrator can't be a plugin). The place it pays off first is **node agents**: many concurrent sessions, lazily spawned, reaped when idle, each with its own MCP servers, wake scope and write fence. That roadmap is a lifecycle problem, and Cordis is a lifecycle runtime with a metatheory behind it.

---

## 2. What Cordis is, in olai's words

The paper names two dimensions of dynamic composability and lifts each from a static type discipline to a runtime mechanism.

| Paper | Cordis API (v4) | olai today |
| --- | --- | --- |
| **Revertible effect** — a context transformation paired with its inverse; the runtime holds the inverses and runs them LIFO on unload (§3.1) | `ctx.effect(() => disposer)`; the body may return one disposer, a promise of one, or a (sync or async) generator that *yields* disposers as setup proceeds. `ctx.on`, `ctx.plugin`, `ctx.provide` and `Service` construction are all effects | none. `PluginServer` has `revision()`, `published()`, `unloaded()` and no teardown; a server half lives for the process |
| **Reactive coeffect** — a plugin declares required keys; every context change is classified activating / deactivating / neutral against the declaration and drives its lifecycle (§3.2) | `export const inject = ['vault', 'deliveries']`. A fiber computes an *epoch* from the uid of each injected service's provider; missing → `PENDING`; a provider replaced (new uid) → unload then reload; a provider gone → unload | `PluginServices` blob; a plugin cannot depend on a plugin; nothing re-resolves |
| **Context** — one object mediating both; a service claims `ctx.<key>`; child contexts derive from parents; property access is a `Proxy` that enforces the declaration at the point of use (§5.1.4) | `class Kolu extends Service { constructor(ctx) { super(ctx, 'kolu') } }` + `declare module 'cordis' { interface Context { kolu: Kolu } }`; `ctx.get('kolu')` for an optional dependency | `PluginServices` (server), `AppFurniture` (browser), both structural blobs re-declared by each plugin |
| **Fiber** — one instantiation of a component, with lifecycle state and a committed view of its dependencies (§4.1, §5.1.3) | `ctx.plugin(fn, config)` → `Fiber`: `PENDING → LOADING → ACTIVE → UNLOADING → DISPOSED`, or `FAILED` with the error on the fiber; `fiber.dispose()`, `restart()`, `update(config)` (runs an `internal/update` waterfall so HMR or a settings page can veto or replace the restart); `ctx.registry` enumerates them | BUILT vs ENABLED, decided once |
| **Isolation** — the same key resolves to a different binding inside a realm (§3.2.3) | `ctx.isolate(key)` derives a child context whose realm table shadows one key; loader `isolate:` on a group | none |
| **Interception** — metadata carried on a context, merged right-biased when a key is *used*, so an enclosing context constrains a component without editing it; changing it triggers no reload (§3.2.3, §6.3) | `ctx.intercept(key, metadata)`; loader `intercept:` | none |
| **Typed events**, five dispatch modes | `emit` (fire and forget), `waterfall` (around-middleware: `(…args, next)`; return without `next()` to short-circuit), `parallel` (`allSettled`), `serial` (in order, first non-nullish wins), `bail`. Listeners are effects, so they unregister with their plugin | ad hoc: `probe()`, `revision()`, `published()`, `unloaded()`, `wake` |
| **Declarative loader** — entries `{id, name, config, disabled, inject, isolate, intercept}`; groups; includes; patches that override a row by `id` or `insert` rows; per-field reconciliation (only `config` changed → `fiber.update`; `disabled` → dispose; `name` → rebuild); `!!js` expressions evaluated against the plugin's own `ctx` (§5.2.1) | three arrays in three files, six edits per new plugin, `--plugins=`, a nix list |
| **Hot module replacement** — changed modules classified by import-graph reachability; stale entries disposed (every disposer runs) and re-mounted from the re-imported module, with rollback if the import throws; no `module.hot.accept` boundaries (§5.2.2) | none |
| **Confluence** (Theorem 80) — whatever sequence of loads, unloads and replacements happened, the quiescent state equals a from-scratch load of the final configuration | why `disabled: true`, patches, `fiber.update` and HMR are safe to reason about | "disabled = absent" holds because the filter runs once; a runtime toggle would have no such guarantee |

Three things the paper says that matter to olai specifically:

- **The boundary is per location, not per medium** (§6.1). A location the system can modify exclusively and restore lies inside and is reverted; a file another program reads or a message already sent is an *emission* that crosses the boundary and is compensated or withheld, never reverted. Vault writes and doorbell deliveries are emissions. Cordis reverts registrations, not the world.
- **A key whose value is a table of independently identified entries is commutative; an ordered chain is not** (§3.4.2). Registries of tools, dressings, kinds, listeners are the first kind, so plugins can be unloaded in any order. Middleware chains are the second, so their order is imposed from outside, by `inject` ordering or `prepend`. This is a design rule for every olai service below.
- **Self-evolving agent harnesses are the motivating example** (§1.2.2) and the stated next validation (§8). dsh already ships it: a model-facing tool defines a versioned package from source, a person approves, the host half mounts as an ordinary fiber, retract is dispose.

## 3. How DeepSeek Harness uses it

Every part of dsh is a plugin, including the model adapter, the tool registry, the session log and the agent loop. "There is no privileged core to patch: you extend dsh by mounting a plugin beside the others, and registrations are effects that unwind when their plugin unloads." The load-bearing shapes:

**A service is a key on `ctx` whose registry methods return disposers.** From `packages/core/tools/src/index.ts`:

```ts
export class ToolRuntime extends Service {
  static inject = ['systemPrompt']
  static Config = z.object({ mode: z.union([...]).default('native'), ... })
  constructor(ctx: Context, config: Config = {}) { super(ctx, 'tools'); ... }
  register(definition: ToolDefinition): () => void {
    return this.layers.effect(this.ctx, layer => layer.tools.insert(name, definition), { label: 'tools.register()' })
  }
}
```

`ctx.tools.register(...)` attaches the disposer to the *calling* plugin's fiber; unloading that plugin unregisters its tools and `tools/change` fires so prompt assembly re-derives. The same shape for `ctx.llm` adapters, `ctx.systemPrompt.section(...)`, `ctx.commands`, `ctx.jobs`, `ctx.fs` providers, `ctx.shell` backends, `ctx.subagent` providers, `ctx.slots` in the browser.

**The smallest plugin is a function with `name`, `inject`, `apply`.** `packages/guard/timeout-policy/src/index.ts`, whole:

```ts
export const name = 'timeout-policy'
export const inject = ['tools']
export function apply(ctx: Context): void {
  ctx.on('tools/execute', async (exec, next) => {
    const timeoutMs = ctx.tools.get(exec.name, exec.agent)?.timeoutMs
    if (timeoutMs === undefined) return next()
    using d = deadline(exec.signal, timeoutMs, TOOL_TIMEOUT)
    const result = await next()
    if (timeoutOf(d.signal, TOOL_TIMEOUT) !== undefined) return toolTimeoutResult(timeoutMs)
    return result
  })
}
```

No registration table, no roster edit. Mount it from a config row and it wraps every tool call; drop the row and it is gone.

**A seam is three roles.** Service Definition (an abstract `Service` declaring the interface, e.g. `abstract class SessionPersistence extends Service` with `create/open/flush/stat/list`), Provider (a concrete subclass mounted from config, `session-persistence-jsonl`), Consumer (a tool or the loop). "Pointing the filesystem and subprocess providers at a remote sandbox moves Bash, PTY and LSP with them, with no provider forks."

**Per-agent scope is a tagged child fiber, not `isolate`.** `packages/core/scope/src/index.ts`:

```ts
export function createScope(ctx: Context, key: ScopeKey): Scope {
  const fiber = ctx.plugin(scope)                       // a no-op plugin; its fiber owns the registrations
  const scoped = fiber.ctx.extend({ [kScope]: key })
  return { ctx: scoped, rawDispose: fiber.dispose, dispose: () => (disposing ??= quiesceFiber(fiber)) }
}
```

and `Agent` does `this.scope = createScope(loopCtx, this); this.ctx = this.scope.ctx.extend({ agent: this })`. Everything an agent's plugins register through `agent.ctx` (tools, prompt sections, listeners) is an effect on that scope's fiber; disposing the agent quiesces the fiber and unwinds all of it. Scoped events (`agent/pre-step`, `tools/pre-execute`) deliver only to listeners registered under that agent's scope chain. `isolate:` realms are layered on top by presets, so two sessions mounting the same preset never collide on `ctx.compaction`.

**The loop is events, and the dispatch mode is part of the contract.**

```text
turn/start
  claim next-step input plus one queued message
  assemble prompt sections + tool schemas
  -> agent/pre-step                   waterfall: reject | enter(messages)
     step/start
     agent/request -> llm/stream -> assistant/chunk* -> assistant/message      (waterfalls)
     tool/call* -> tools/pre-execute -> tools/execute -> tools/post-execute -> tool/result*
     step/end
  -> agent/turn-stopping              serial
turn/end
```

Compaction, plan mode, Claude-Code-style hooks, a repeat-tool guard, time context and tmux context are each a plugin listening on one of these. Durable facts (`turn/*`, `user/message`, `assistant/*`, `tool/*`) go to the session log; **model-visible means logged** is asserted at runtime. (olai's replay-misattribution RCA in this vault is a violation of exactly that invariant.)

**Composition is layered config.** A *profile* (`web`, `headless`, `sdk`, `acp`) stacks *bundles*, each a `cordis.patch.yml` of rows plus the code they mount, then the user's patch, then `--patch` overlays. A patch is `- id: typert-loader\n  disabled: true` or an `insert:` block. `dsh --profile web --dump-config` prints the tree the machine boots; `dsh plugin` installs an out-of-tree package into a profile. From `presets/standard/agent.cordis.yml`:

```yaml
- id: tool-bash
  name: '@deepseek-ai/dsh-tool-bash'
  disabled: !!js process.platform === 'win32'
- id: planning
  name: cordis:group
  group: true
  isolate: { planMode: true }
  config:
    - id: plan-mode
      name: '@deepseek-ai/dsh-plan-mode'
```

## 4. What olai gets

### 4.1 Teardown, therefore runtime enable, failure containment, reload

Today "disabled" means the entry was never added before composition. That works once and cannot express: turn odu off for this serve without a restart; a plugin whose `serve()` throws (padi socket gone at boot) staying failed while the rest run; a plugin re-reading its vault config file without a restart; a developer editing `olai-plugin-kolu` and seeing it re-mount.

With every registration an effect, all four are `fiber.dispose()` and `ctx.plugin(...)` again, and the loader performs them from a config edit. A plugin that throws during `apply` lands in `FAILED` having installed nothing (Corollary 69), siblings untouched. The property olai's own doc calls the design's best, "disabled is a state the framework's composition already expresses", now holds at every moment instead of at boot.

### 4.2 Services, not a blob; plugins that need each other

`PluginServices` is seven fields and growing, built per plugin by hand in `runtime.ts` (`doorFor(plugin.name)`), and every plugin receives all of it. Under Cordis each field is a service a plugin names:

```ts
// olai-plugin-odu/src/server.ts
export const name = 'odu'
export const inject = ['vault', 'deliveries', 'kinds', 'surfaces', 'wakes']

export function apply(ctx: Context) {
  ctx.kinds.register({ kind: 'worktree', takes, admits: isPathShaped })   // the service composes odu-worktree from ctx.fiber.name
  ctx.surfaces.register(surface, oduHalf(...), faces)                       // sibling under the fiber's name
  ctx.vault.revision(snapshot => doorbell.consider(snapshot))            // replaces PluginServer.revision; a contained door, not a bare emit
  ctx.wakes.register({ subject, from, waiting, kinds, faults })
}
```

The stamp that today is threaded by the root ("core marks that row with the plugin's name, stamped from the registry binding and never from the caller") becomes the fiber's name read by the `deliveries` service at registration, the same guarantee with no threading. A plugin that does not name `deliveries` never sees it. And a plugin may name **another plugin's service**: the in-flight `plugin-spaces-mirror` lane wants kolu's fleet and odu's runs, an edge the current design can only wire by hand at the root. With `inject = ['kolu', 'odu']` it waits for both, unloads when either leaves, and touches nothing in core. Cycles are not a runtime hazard: two plugins each declaring the other's key never activate, and the runtime reports it at load from the declarations alone (§6.5).

### 4.3 Chat as events; the doorbell as a waterfall

`probe()` must answer both halves at once, an invariant with an incident behind it (asking twice started a daemon twice). A waterfall gives that invariant for free, one dispatch per session start:

```ts
// core
const start = await ctx.waterfall('chat/session-start', { servers: [], missing: [] }, s => s)
// olai-plugin-kolu
ctx.on('chat/session-start', async (start, next) => {
  const probed = await probe(ctx.env)
  probed.server ? start.servers.push(probed.server) : start.missing.push(probed.missing)
  return next()
})
```

Delivery policy (idle takes it as a turn; busy holds; nobody there queues) is a `chat/deliver` waterfall whose short-circuiting listener is core's policy plugin, `coalesce` a listener before it. Each rule becomes a plugin that can be read, replaced or tested alone, which is how dsh ships compaction and plan mode.

### 4.4 Node agents are scopes

Node agents are on master (phase 1 of that plan: the roster over `prop:agent`, the door on agent rows, the panel switching, the subtree-memory teaching). What is not on master is concurrency: `agent.ts` still holds one `session: string | null`, so an agent that is not the open one has no process. The plan's phase 2 says "hold a map", and its later phases say sessions spawn lazily, reap when idle, are capped, and each carries its own MCP servers, its subtree as wake scope, and a write rule ("an agent writes only strictly inside its own subtree and asks its ancestor for anything above").

dsh's `createScope` is the exact shape:

```ts
// core: node agents
const scope = createScope(ctx, node)                    // a child fiber; dispose = every registration gone
const agentCtx = scope.ctx.extend({ node })
agentCtx.intercept('vault', { writable: subtreeOf(node) }) // the write fence, carried on the context
agentCtx.plugin(session, { engine: node.props.agent })    // probes run here; MCP servers register here
```

Reaping an idle session is `scope.dispose()`; a wake for a session with no scope creates one (the paper's "a deactivation may chain straight back into an activation"). The subtree write rule is **interception metadata**: consulted by the vault service when a write is attempted, installed by the enclosing context, invisible to the session's code, and adjustable at runtime without a reload (§6.3 uses precisely this example: read-only access for a community component, full for a core one). Two node agents on different engines are two providers under two `isolate` realms of the `engine` key. The cap and the lazy spawn are one policy plugin over `ctx.registry`. Concurrency stops being a rewrite of `agent.ts` and becomes the default shape, which is what the human said phase 2 requires ("the ceiling is ours, not the protocol's").

### 4.5 A config tree, not three rosters

Adding a plugin is six edits in `plugin-api` today, three of them arrays that must agree in order. Under the loader the build's set is a bundle:

```yaml
# packages/bundle/base/olai.yml
- id: vault
  name: '@olai/store'
- id: kinds
  name: '@olai/format/kinds'
- id: chat
  name: '@olai/chat'
- id: kolu
  name: 'olai-plugin-kolu'
- id: odu
  name: 'olai-plugin-odu'
```

`olai web`, `olai surface` (headless MCP) and `olai orchestrate` are profiles stacking this bundle with their own. `--plugins=odu` and the nix option write a patch (`- id: kolu\n  disabled: true`). `olai --dump-config` shows an operator the tree this serve boots. A third party runs `olai plugin add olai-plugin-saatchi` and the row lands in their profile: **no rebuild**.

### 4.6 Slots, not four browser registries

`dressings.ts`, `Chrome.tsx`, `Mounted.tsx`, `marks.ts` are four hand-written walks over `ROSTER`. The browser runs its own Cordis app (Koishi's console and dsh's web client both do) and `ctx.slots` is one typed registry: an owner declares a slot (`outline.row.block`, `app.header`, `chat.speaker.mark`) with a cardinality (`single`, `list`, `keyed`, `chain`) and a scope; a plugin registers into it and never imports another feature's component; disposing the plugin removes its cells; disposing the owner collapses its children. The licence consult stays exactly where it is (the server answers the *word*; the browser looks it up) and the lookup is the `keyed` slot `outline.row.dressing`.

### 4.7 Plugins an agent writes

dsh's extensions subsystem: `cordis_define` stores a versioned package from source; `cordis_run` asks the person; the host half mounts as a fiber owned by the session; `cordis_inspect` lets the agent query the live registry and slot tree before writing code; retract is dispose. For olai: a node agent whose subtree holds a small dressing or doorbell and whose olai mounts it, live, under the same teardown as kolu. It is the paper's stated future and the reason not to build a second runtime later.

## 5. What it costs, honestly

**Two composition systems on the server.** olai's server is Effect (`effect@4`); Cordis is a lifecycle runtime over plain host code. They are not rivals for one job. Effect's `Layer` is static composition, built once, with no re-resolution when a provider leaves; Effect's `Scope` is the revertible half without the reactive half. The paper says Cordis is "an overlay, realizable atop a language of either style" (§7.2). The split that works: Cordis for *what is mounted and when*, Effect for *how a mounted thing does its work*. The alternative, re-implementing reactive coeffects and the loader over Effect's `Scope`, is the paper's Section 5 written again without its proofs.

**How Cordis arrives: an npins pin, never a copy.** dsh vendors Cordis into `vendor/` under its own `@deepseek-ai` scope with a sync script. olai does not copy libraries into the repo; this machine's rule, and the repo's standing law, is that the app requires no dependency outside Nix itself. So Cordis comes in the way `@kolu/*` and `@odu/run-client` do: a pin in `npins/sources.json` on `cordiverse/cordis` (upstream, not dsh's rescoped fork, so `import { Context } from 'cordis'` and `declare module 'cordis'` read as written), a `nix/cordis.nix` beside `nix/kolu.nix` and `nix/odu.nix` naming the packages olai's source seeds (`packages/core` as `cordis`, plus `loader`, `include`, `group`, `hmr` as `@cordisjs/plugin-*`) as the `(src, dest)` pairs kolu's `hydrate-kolu-packages.sh` copies into `node_modules` as raw TypeScript, and the pin's closure (`cosmokit`, `schemastery`) riding the same pin or its own. `scripts/check-hydrated-deps.sh` asks its one question of the new pin exactly as it does of the other two: every external the hydrated sources declare is in olai's root manifest at the pin's version, so there is one `cordis` and one `effect`, never two. `just update-pins` walks it forward with everything else, and `npins/sources.json` records the revision, so what the tree compiled against is always in the diff. The API is marked "not yet stable" upstream, which is an argument for the pin, not against it: a moved revision shows as a diff and a typecheck, never as a silent drift inside a copied directory.

**The wire is composed once today, and the spike confirmed it.** `implementSurfaces` and `mergeDisjointGroups` are one-shot over the map they are handed; there is no incremental add or drop on a live group. A `ctx.surfaces` that re-calls them on register/dispose works (sibling tags appear under `surface/<name>/`, dispose restores core's tags exactly), but it re-implements every surviving sibling each time, state survives only because it lives in each plugin's `deps`, and a server holding the previous fused handlers keeps serving them. On the client, `connectSurfaces` takes `surfaces` at the call and `SurfacesConnection` has `dispose` and no `update`, so a roster change is a reconnect. This is a framework ask to `@kolu/surface` and `@kolu/surface-app`, filed before phase 2 starts: live add/drop on a rooted bundle plus `update(surfaces)` on a live connection, or reconnect-per-roster-change documented as the contract. dsh solves the analogous problem with `tools/change` and per-step re-derivation.

**Typing moves from `satisfies` at the registry to declaration merging.** `ctx.kolu` is typed by `declare module 'cordis' { interface Context { kolu: Kolu } }` in kolu's own package; a consumer gets it by `import type {} from 'olai-plugin-kolu/server'`. As strong as `as const satisfies`, more local.

**Async disposers run concurrently** (reverse registration order, but async ones in parallel). Ordered teardown lives inside one effect, ideally as a generator yielding disposers in sequence. Import dsh's defensive-patterns rules with the code: dispose must reach quiescence; contain listener exceptions in the dispatcher; report orthogonal outcomes independently.

**Reversion is bounded by the boundary.** Vault writes, delivered sentences, dialed sockets are emissions; the runtime reverts registrations. Not worse than today, where nothing is reverted, but not undo for vault edits.

**Dependency identity is nominal.** A key is a string; drift between independently built providers and consumers is caught by peer-dependency ranges, not the runtime (§6.6). Name services by package (`kolu`, `odu`, or `kolu.fleet`) the way kinds are prefixed today.

**HMR on Bun: no.** dsh runs HMR on Node under `tsx`, deleting Node's internal ESM `loadCache` entries and `require.cache`, then re-importing the same specifier. Under Bun 1.4 `node:internal/modules/esm/loader` does not exist, a rewritten file at the same URL is served stale, a query string on that URL is also stale, and the old `Loader.registry` global is gone. Only a different file path yields a new evaluation. HMR in dev is off the plan until a Bun-specific cache bust exists.

**Two pin frictions, both throwaway in the spike and owed by phase 2.** Cordis's `package.json` points at `lib/`, so `nix/cordis.nix` rewrites `exports["."]` to `src/` and stamps every file `@ts-nocheck` because olai's `tsc` is stricter than Cordis's. Phase 2 either typechecks the pin under olai's `tsc` (the raw-TS argument the kolu and odu pins already make) or files the strictness delta upstream, and the rewrite must cover every subpath the code imports. Separately, Cordis exports `FiberState` as a `const enum`, which `isolatedModules` cannot consume as a value; the spike pins the numbers locally, and the upstream ask is a regular enum or a const object.

## 6. The plan

Each phase is one PR, green alone.

1. **Spike: done** ([juspay/olai#472](https://github.com/juspay/olai/pull/472), draft, unmerged, nothing in `packages/cordis-spike` ships). The `cordis` pin is in `npins/sources.json`, `nix/cordis.nix` seeds `cordis` core and `@cordisjs/plugin-loader` through kolu's copier, and `just cordis-deps` asks the same question of it as of the other two pins. On real tenants: `disabled: true` at runtime leaves core's tags byte-identical; a throwing `apply` lands `FAILED` with kolu `ACTIVE`; `--plugins` is a `disabled` patch the loader applies at mount and later; the Proxy `ctx` reads fine inside `Effect.gen`; `ctx.intercept` is read through `Service.resolveConfig` on the derived context with no reload. Its answers are folded into §5 and §8, and its five review findings are the constraints on phase 2 below.

2. **The app on Cordis, whole: done** ([juspay/olai#474](https://github.com/juspay/olai/pull/474), merged 2026-09-03 as `35f7fb9d`). One lane, in this order, each step green alone, and the lane merges as one unit or not at all. *Ruled by the human, 2026-09-02: olai is one app, not a client and a server; a PR that finishes the server half and covers the seam with a test does not merge, and a hand-written list held equal to a data file by a test is duplication, not a fix.* The server steps below landed on the branch first; the browser steps were pulled forward from what was phase 5 and belong to the same unit.
   - **File the framework ask first** (§5): live sibling add/drop on a rooted bundle and `update(surfaces)` on a live `connectSurfaces` connection, or reconnect-per-roster-change as the documented contract. If the ask is slow, phase 2 starts with reconnect-on-change and says so.
   - **Re-compose moves into the composition root** (`packages/server/src/runtime.ts`) before anything else moves. The spike's `Surfaces.recompose` re-implements every surviving sibling; the root's version must compose existing runtimes, set the real `plugins` cell in `@olai/surface` (the spike only shape-proved its roster against `PluginRoster`), and swap the served handlers so a live connection does not keep the old fused group.
   - **`PluginServices` dissolves into services** (`vault`, `env`, `clock`, `log`, `deliveries`, `kinds`, `surfaces`, `wakes`), each a `Service` whose registration methods return disposers attached to the calling fiber; the per-plugin stamp comes from `ctx.fiber.name`.
   - **Hooks become doors and one waterfall.** `revision` and `unloaded` became contained subscriptions on `ctx.vault` (`vault.revision(handler)`, `vault.unloaded(handler)`), not bare Cordis events: Cordis's `emit` is a loop with no `try` in it, so a throwing listener would have silenced every later plugin and failed the directory fiber; the door wraps each handler once and warns with the calling fiber's name (the shape `ctx.watching.subscribe` already had). `probe` becomes the `chat/session-start` waterfall (§4.3). `surfaces/published` was declared and never emitted, so it went; the roster cell carries what it would have said. **`unloaded()` is not teardown**: it means the store failed to publish. A half that has teardown beyond its effects returns a disposer from `apply` (xyne-spaces is the first caller).
   - **The three rosters become the base bundle**; `--plugins` and the nix option write a `disabled` patch through include's `applyEntryPatches` (the spike's `pluginsPatch` was the same data as a function); `plugin-api` shrinks to service definitions; `fence.test.ts` keeps every claim except plugin-imports-plugin, and the `cordis` arm in its `FRAMEWORK` regex becomes permanent.
   - **Pin hygiene rides along**: `@cordisjs/plugin-include` and `-group` join the pin; the `@ts-nocheck` stamp goes (typecheck the pin under olai's `tsc` or file the delta upstream); the `exports` rewrite covers every imported subpath; `FiberState` is filed upstream as a regular enum or const object and the local numbering is deleted.
   - **One roster, end to end.** `olai.yml` is the only place a plugin is named. The tab's plugin table (today's hand-written `WIRES` and `PLUGINS`) is generated from that file at build time by `packages/web/src/build.ts` or a step beside it. A third plugin is one row. `rosters.test.ts` is deleted, because there is nothing left to hold equal.
   - **The tab is a Cordis app.** A client `Context` in `packages/web/src/client` with a `slots` service (declared slots with cardinality: `outline.row.block`, `outline.row.chip`, `outline.row.pane`, `app.header`, `app.drawer`, `app.mount`, `chat.speaker.mark`). The four walks over the manifest list (`live/dressings.ts`, `Chrome.tsx`, `Mounted.tsx`, `plugins/marks.ts`) become slot reads; each plugin's browser half becomes `apply(ctx)` registering into slots through `ctx.effect`; `AppFurniture` becomes services the browser half names in `inject`, the move `PluginServices` made on the server.
   - **The tab follows the roster.** Dial core first, read the `plugins` cell, bring in the siblings it names through `SurfacesConnection.redial(surfaces)` (kolu#2223's client half). A plugin turned off on the server leaves the tab without a reload. "Mount assumes OFF until the roster lands" becomes the boot order rather than a guess. The `plugins` cell gets an `equals` so a no-op re-compose does not redial.
   - **Disabled is absent in the tab too.** The bundle ships every built plugin's code, but per-plugin code splitting (a dynamic `import()` per row, generated with the table) means the tab loads and evaluates only the plugins the roster names.
   - **Not started in this lane**: HMR (no Bun cache bust exists, §5); intercept in the store (it belongs on the `vault` service, §8); node-agent scopes (phase 6); the kind vocabulary's boot-time snapshot in `propKinds.ts` (phase 7's debt). Retire `.worktrees/cordis-spike` when the draft closes.
3. **One directory per plugin, named by the plugin: done** ([juspay/olai#480](https://github.com/juspay/olai/pull/480), merged 2026-09-03 as `bc2f322f`, a rename diff with no deferrals). As landed: `packages/plugins/kolu/`, `odu/`, `xyne-spaces/`; the two client packages deleted and folded in behind each tenant's `./appliance` export (kolu's dial sits at `src/client/` because `src/appliance/` already held its UI fold; odu's at `src/appliance/`); `@olai/tests` reaches the fake padi through `olai-plugin-kolu/appliance/testlib`, recorded as two files and one declaration rather than a general-package exception. The brief it landed against: The tenants move to `packages/plugins/kolu/`, `packages/plugins/odu/`, `packages/plugins/xyne-spaces/` (the human, 2026-09-02): the container already says "plugin", and everywhere else the plugin is one word (the row's `id`, `--plugins=kolu`, `docs/plugins/kolu.md`, the sibling key on the wire), so the directory is that word. The package *name* stays `olai-plugin-<name>`: it is the module specifier the row mounts and the shape an out-of-tree plugin would publish under, and bun's workspace finds a package by its manifest, not its folder. `packages/kolu-client` and `packages/odu-client` fold into those directories as `src/appliance/` (or a `./appliance` subpath if a second door is needed) and the two packages are deleted. They exist because, before phase 2, a plugin could not import the interface and the only way to get an appliance out of core was a package that depended on nothing; that reason is gone. They are also the one place the fence's claim is bent by naming rather than by import. xyne-spaces, which arrived after the runtime, is one directory and is the shape. `fence.test.ts`, `prove-fence.sh`'s container derivation, and `check-hydrated-deps.sh` follow the move.
   - **Done means**: the three tenants at `packages/plugins/<name>/`, `kolu-client` and `odu-client` deleted, and `fence.test.ts`, `prove-fence.sh` and `check-hydrated-deps.sh` green on the new paths. Nothing in the runtime or the plugin API changes.
4. **The plugin API is Effect: done** ([juspay/olai#483](https://github.com/juspay/olai/pull/483), merged 2026-09-03 as `31f85927`; the human's own third review round folded before the merge). One PR, a content diff on every plugin's `server.ts`, reviewed on the final paths phase 3 gave them, and before the agents lane writes three more plugins against whatever API exists then. *The human, 2026-09-02: olai uses Effect; for an architecture with no hacks or escape hatches, plugins are written in Effect and Cordis is an engine nobody outside one package sees.*
   - **Why.** After phase 2 the two runtimes meet through bridges in both directions: the composition root wraps Cordis calls in `Effect.promise`, Cordis services call back into Effect through `ring(Effect.log…)`, and plugin bodies are plain TypeScript that reach into Effect code by hand. Each is an escape hatch and every new plugin copies them. Effect's `Scope` (finalizers, LIFO) already is the paper's accumulator; its requirement channel already is the paper's coeffect specification. Two vocabularies for one idea is the duplication this lane removes.
   - **A plugin's `apply` is an Effect**: `(ctx) => Effect<void, never, Scope | Vault | Kinds | Surfaces | …>`. Its revertible effects are `Effect.acquireRelease` and `Effect.addFinalizer` on that `Scope`; a facade in `@olai/plugin-api` runs the Effect inside the Cordis fiber and closes the scope from the fiber's disposer. Authors never call `ctx.effect`.
   - **`inject` is the requirement channel.** The services a plugin needs are the `R` of its `apply`. Because the compiler cannot hand Cordis a runtime list from a type, the plugin declares the list once beside `apply`, typed so the two cannot disagree (a helper that takes the Tags and yields both the list and the `R`). A mismatch is a `tsc` error naming the plugin, the same shape `as const satisfies` gave the old registry.
   - **Services are Effect services.** `ctx.vault`, `ctx.kinds`, `ctx.surfaces`, `ctx.deliveries`, `ctx.wakes`, `ctx.watching`, `ctx.held`, `ctx.env`, `ctx.clock`, `ctx.log` become `Context.Tag`s whose implementations are provided by the facade from the Cordis-side services; `ctx.log` is Effect's logger; `deliver` and `register` return Effects; a registration's undo is a finalizer on the calling plugin's scope. The vault doors become `Stream`s (`vault.revisions`, `vault.unloaded`) and the waterfall (`chat/session-start`) becomes Effect middleware. The stamp still comes from the fiber: the facade reads `ctx.fiber.name` once and provides it as the plugin's identity.
   - **Cordis is confined to one package, `packages/effect-cordis`.** The translation lives there and nothing else does: `definePlugin(effect)` (open a `Scope` on activate, close it on dispose, a throwing Effect lands `FAILED` having installed nothing), the requirement-to-`inject` helper, `serviceTag(name)` (a Cordis key as an Effect `Tag`, provided from the fiber's context), `eventStream` and `waterfall` (events as `Stream`s and Effect middleware), and the loader entry as an Effect. It has no olai noun in it and is tested with toy services: a plugin sits `PENDING` until its `Tag` is provided, finalizers run in reverse on dispose, a replaced provider re-runs the plugin, a `Stream` ends when the plugin unloads. `@olai/plugin-api` holds olai's `Tag`s and event names and imports `effect-cordis`, never `cordis`; no other package imports either, and the fence's `cordis` arm narrows to `effect-cordis` alone. This is also where Cordis's instability has its one home: a pin bump that moves the API changes this package and nothing else, and the next-gen track (§9) mounts the core's own parts through it without going through the plugin interface. It stays translation: the moment it grows an opinion about what a plugin is, that opinion belongs in `plugin-api`. The loader still mounts a module by name from a row; what the row points at is the plugin's `server.ts` export wrapped by `definePlugin`. Cordis stays the engine for the reactive half and the loader, the part worth not writing again; the Effect-native alternative stays named as the other consistent answer and is not taken.
   - **Done means**: three plugins written as Effects with no `cordis` import; the composition root's `Effect.promise` wrappers and the services' `ring(...)` callbacks gone; the bridge is `packages/effect-cordis`, with its own tests; the fence's `cordis` arm narrowed to that package and prove-fence catching a plugin that imports `cordis` directly.
5. **Agents as plugins** (in progress; one PR, dispatched 2026-09-03 to claude-opus on `agents-as-plugins`, manual merge). One plugin per engine, never a roster plugin: the three rows share no release clock (Claude's adapter pin moved five times in a month, opencode's zero), `--plugins` enables them one at a time, and the fence gets one more general package that spells no engine.
   - **What exists (on `plugin-api-effect`).** `packages/chat/src/agents/{claude,opencode,pi}.ts` are the three *legs* (each engine's wire bets: `_meta` keys, tool spelling, steering, queue advertisement) over the `Leg` interface in `leg.ts`; `roster.ts` holds a hardcoded `KINDS` (`id`, `leg`, `at(where) => Adapter | null`) and `rosterOf(where)` over it; `models.ts` carries two Claude-only bets (the picker id `model`, the alias tiers); `@olai/surface/chat.ts` exports `AGENTS = { claude, opencode, pi } as const` and `AgentId = keyof typeof AGENTS`, a closed union that makes a fourth engine a core PR; `acp/patches/` and `nix/acp-agent.nix` hold the adapter pin and its patches for all three; the no-agent face's install sentences live in `@olai/web`. The engine's MCP servers reach a session through the `SessionStart` waterfall (`plugin-api/src/services.ts`), which still carries two things phase 4 left by name: a plugin pushes `{ name, ask }` and signs its own `name`, and `ask` answers a `Promise<Probed>` where everything else a plugin hands over is an Effect.
   - **The shape.** `packages/plugins/claude/`, `odu`-style: one row each in `olai.yml` (enabled by default; `xyne-spaces` is the opt-in precedent). A new tag `Agents` in `plugin-api/src/services.ts`, provided from a `registry` keyed by the fiber's name: `agents.register({ leg, at, missing, prompt })` where `id` is the fiber's word (the plugin cannot spell another's), `leg` is the engine's `Leg`, `at` finds it on this host (`Adapter | null`; `null` is "not installed", not a fault), `missing` is the install sentence (`NotHere | null`, the plugin's whole sentence, `null` for a shipped-in engine), and `prompt` is the CHANNEL the standing prompt rides (`_meta` vs first turn). The system-prompt TEXT stays one core module versioned with the binary. Each engine's adapter pin and patches travel into its plugin directory (the 0.66.0 patch debt goes with Claude); `nix/acp-agent.nix` splits per plugin or is parameterised by the row. `models.ts`'s two Claude bets move to `plugins/claude`. `@olai/acp` stays a general package: the protocol is the language, not an integration.
   - **Core, before → after.** `rosterOf(where)` reads the `Agents` registry instead of `KINDS`; `AGENTS` becomes data the server sends (an `engines` list beside the existing roster, or a cell), `AgentId` becomes `string`, and every table keyed by it follows; the picker draws what arrived; the no-agent face's rows are each enabled plugin's `missing.why`. A node's `agent` property names an engine id and resolves against the registry; an id no enabled plugin registered is the same absence a missing binary is. `OLAI_ACP_AGENT=""` keeps its meaning; `--plugins=opencode,pi` serves a panel with no Claude row, no probe, no mark.
   - **The session-start door, finished.** The collector keys `asking` by the fiber that pushed, so a plugin pushes `{ ask }` and the name is stamped like every other door; `ask` answers an Effect, and `@olai/chat` runs the thunks as Effects under its bounded concurrency (`AT_ONCE`) rather than as promises. Both are edits on the session-open path this lane already touches.
   - **Done means**: `KINDS` deleted; `AGENTS` no longer a union and no general package spells an engine (fence claim, with a prove-fence mutation); three rows; `acp/patches` split per plugin; `SessionStart` carries no `name` and no promise; every scenario in `packages/tests/features/` that opens a session passes unchanged; the plugins panel draws the three engine rows with the same five states as any plugin.
   - **Not started**: node-agent concurrency (phase 6); dissolving `@olai/acp`; a fourth engine (the point is that it would be one directory and one row, and nothing here should need it to prove that).
6. **Node agents as scopes** (one PR, after phase 5). Delivers node-agents phase 2 (concurrency) and phase 4 (derived wakes) as lifecycle rather than as a map in `agent.ts`; node-agents phase 1 (the roster over `prop:agent`, the door on agent rows, the panel switching, the subtree-memory teaching) is on master.
   - **What exists.** `packages/chat/src/agent.ts:407` holds `let session: string | null`: one live session, so an agent that is not the open one has no process and a wake for it is held, not acted on. The roster's live states are true for one row at a time. Wake scope is a manual pick per conversation, kept in `@olai/state`'s `wake` record and served through `chat.scope` (`packages/chat/src/scopes.ts`); a plugin's doorbell asks `deliveries.scopes()` and gets `(agent, session, file)` triples. `Held` is per plugin per vault. The pinned ACP adapter already holds many sessions keyed by id and tears one down at a time, so the ceiling is olai's.
   - **The shape.** In `@olai/chat`, a node agent is a `Scope` and a session is a fiber under it, opened by `Effect.acquireRelease` and closed by the scope: spawn lazily on the first message or the first wake in scope, reap on an idle deadline, cap the live count (policy in one place, over the registry, benched). A session's ACP process, its MCP servers (the `SessionStart` answer), its inbox and its delivery door are all acquired on that scope, so reaping is one `Scope.close` and nothing is remembered. The panel routes turns and deliveries by node, not by "the" conversation; the roster's states are computed per row from the live map.
   - **The write fence.** The node-agents rule "an agent writes only strictly inside its own subtree and asks its ancestor for anything above" is a per-session `Provision` of the vault's write door narrowed to the subtree, installed by the node agent's scope and invisible to the session: the same shape `Deliveries` already has (a door minted per plugin closing over the word). Enforced in `@olai/ops`'s write gate against the session's identity, not in the prompt.
   - **Derived wakes.** For a session bound to a node, `deliveries.scopes()` derives `file` from the node's subtree (the outline that holds it), so kolu's and odu's doorbells need no change; the manual picker survives only for unassigned conversations. An agentless wake climbs to the nearest ancestor node agent, and the orchestrator ends up the root catcher. `@olai/state`'s `wake` record keeps only the picks for unassigned conversations.
   - **Done means**: `let session` gone; two node agents converse at once, both roster rows live, in a scenario in `node_agents.feature`; a wake for an agent that is not on screen is acted on; an idle session is reaped and its next message respawns it with the subtree as memory; the picker is absent on an agent-bound conversation; a write outside the subtree is refused with the ancestor named.
   - **Not started**: the Unassigned migration UI and distillation turn (node-agents phase 3); agency (an agent creating child nodes and seating sessions, phase 5 of that plan); the Spaces binding; which engines can hold many sessions per directory cleanly (a fact-check this lane records per engine as it goes).
7. **Loader surface** (one PR). What an operator can see and do with the rows, now that the runtime can move them.
   - **What exists.** `--plugins` is a `disabled` patch over `olai.yml` applied by `@cordisjs/plugin-include` at boot (`packages/bundle/src/bundle.ts`, `mountRows`); `reportBundle` reads each row's fiber state; the `plugins` cell carries the roster with an `equals`, and the tab follows it through `redial`. The nix home module writes the same flag (`nix/home/module.nix`). The plugins panel (a door in the bar, the human's ask) draws five states per row and is read-only by a standing decision: enablement is the instance's, CLI/nix only, no verb a press could call. The kind vocabulary is read once at boot (`packages/server/src/propKinds.ts`, "READ ONCE, at boot, and that is a phase boundary"): the store's codec holds it for the life of the process, so a plugin that unloaded mid-serve would leave the vault validating against a word nobody claims.
   - **`olai --dump-config`.** Print the rows after every layer: the file's own `disabled`, then the flag or nix overlay, then the live state (`running`, `waiting`, `failed` with the plugin's own sentence, `off`), which is the same table the panel draws and the honest form of "which of the two am I looking at". Implemented over `ROWS`, `pluginsPatch` and `reportBundle`; no new source of truth.
   - **The kind vocabulary follows the fibers.** `propKinds` becomes a live reading: the store's codec asks the `Kinds` registry at validation time (or is told on `changed`), so a row that leaves takes its words with it and a row that arrives brings them. This is the one piece of core that phase 2 left as a boot-time snapshot and that a live toggle needs; bench it with a plugin disposed after the store opened.
   - **A live toggle, as capability first.** The host can already dispose and re-mount a row (`mountPlugin`, `Mounted.dispose`); expose that as `loader.update(id, { disabled })` on the bundle door and bench it end to end (a disabled row leaves the wire, the roster cell moves, the tab redials, the kinds follow). Who may call it is a separate line: the standing decision says the panel gets no verb, so the caller is an operator verb (`olai plugins enable|disable <name>` against the running serve, or a nix change plus SIGHUP re-read of the flag). **Decision for the human**: whether the panel ever gets a switch. The lane ships the capability and the operator verb and leaves the panel read-only unless told otherwise.
   - **Out-of-tree plugins.** A row may name a package outside the workspace, and the server half loads at runtime through the loader's `resolve`. The browser half cannot: a tab's chunks are generated at build from `olai.yml`. So "add a plugin" in a Nix world is a nix option (`services.olai.extraPlugins`, each an npins pin plus a row appended to the bundle at build) and a rebuild, not a runtime installer. `olai plugin add` is dropped from the plan; the option and its check in `nix/home/check.nix` replace it.
   - **Done means**: `--dump-config` prints the layered table; a plugin disposed after boot leaves the vocabulary and the wire, and the tab follows, in one scenario; the operator verb exists and is documented in `docs/running.md`; `extraPlugins` builds a binary with a fourth row from a pin.
   - **Not started**: HMR (no Bun cache bust exists, §5); a settings file (the configuration ruling of 2026-09-02 puts configuration in `.olai` files, and enablement is the instance's, so there is none).
8. **The orchestrator as a profile** (see §7): `olai orchestrate` stacks the base bundle with a driver group (wake listener, gate policy, dispatch tools) over the same context, the way dsh's agent loop is a plugin.
9. **Dynamic plugins** (one PR, after phase 7). A node agent writes a plugin into its subtree, a person approves it, olai mounts it while serving, and retract is dispose. The paper's stated next validation, and dsh's `cordis-host-runner` (`cordis_define` / `cordis_run` / `cordis_inspect`) is the reference.
   - **What exists.** Every mechanism: a row is a module specifier the loader `import()`s through `resolve` (`packages/effect-cordis/src/loader.ts`), and Bun imports TypeScript from disk; a plugin is `definePlugin` over Effect with `needs`; `mountPlugin` and `Mounted.dispose` mount and unmount one; the roster cell moves and the tab redials; the plugins panel is where a person looks. What does not exist: a place for the source, an approval, and a browser half that was not built.
   - **Where the source lives.** In the vault, under the node agent's subtree, as the configuration ruling requires: a node with a `plugin` property whose subtree holds `server.ts` and, optionally, `browser.tsx`, written by the agent through the ordinary vault write door (so the subtree fence of phase 6 applies, and so the plugin is versioned by the ledger like any other file). The row is derived: `id` is the node's word, `name` is the file's path.
   - **The tools (agent face).** `plugins.define` writes or replaces the files and returns the row's identity and the version (a content hash); `plugins.run` requests activation of one version; `plugins.inspect` answers what a plugin may name (the tags, the slot catalog with cardinality and current occupants, the roster) so the agent reads the registry before writing code; `plugins.stop` and `plugins.undefine` retract. Each is a member on core's surface exposed to the agent face, and each is a durable fact in the conversation the way a tool call is.
   - **The approval (browser face).** A `plugins.run` from an agent lands as a pending row in the plugins panel with the source visible; a person approves once, or approves the plugin for later versions too; the panel is where the read-only rule gets its one verb, because this is the case where the fact being decided is a person's, not the instance's. Until approved, nothing mounts. This is the paper's §6.3 read honestly: the code runs with the process's authority, so approval by the owner is the boundary, and sandboxing is out of scope.
   - **Mounting.** Server half: a row appended to the live bundle (`loader.update` from phase 7), so it is a fiber like any other, with the same states and the same containment. Browser half: the server builds the file with `Bun.build` into a chunk under `/_olai/plugins/<id>-<hash>.js` and the roster cell carries the chunk URL beside the row; the tab's `BROWSER_ROWS` becomes the generated table joined with the dynamic rows from the cell, and the existing redial mounts it. A version switch is dispose-then-mount; a render failure in the tab is reported back and the row shows it.
   - **Done means**: an agent defines a dressing for a kind it claims into its subtree, the person approves, the chip appears in the tab without a reload, `undefine` removes it and the chip is gone; a failed server half lands `failed` with the plugin's sentence and touches no sibling; every define is a ledger commit like any vault write.
   - **Not started**: sandboxing or capability attenuation for agent-written code (the paper's `intercept` is the seam, and phase 6's write fence is the first use of it); publishing a dynamic plugin as a package; anything the orchestrator profile (phase 8) would need beyond this.

## 7. olai's rulings, re-tried

Each earlier ruling, and what a fresh look says. "Keep" means it survives on merit, not by precedent.

| Ruling (source) | Verdict | Why |
| --- | --- | --- |
| Compiled-in registry; "the boundary is the value, not the loading"; a third party rebuilds olai (`plugin-system.md` §What compiled-in still cannot do) | **Overturn** | The loader mounts a package by name from a config row; `olai plugin add` is the whole story. Compiled-in was the cost of having no runtime, not a design. |
| The registry never grows a dependency arm; no plugin consumes a plugin; the root threads capabilities (eighth sitting) | **Overturn** | `inject` is the dependency arm, and it is reactive: the "half-wired state" the ruling feared is exactly `PENDING`, a legitimate, inspectable state with a theorem about when it resolves. `plugin-spaces-mirror` is the first edge that needs it. |
| Enable is CLI/nix only; preferences rows are read-only (#434's shape) | **Overturn** | The row is data on a config tree; a toggle writes `disabled` and the loader reconciles. The honest "who said so" survives as which layer carries the override, shown by `--dump-config`. |
| Three rosters (`WIRES`, `SERVERS`, `PLUGINS`) must agree; six edits per plugin | **Overturn** | One config row. The three *doors* (wire, server, browser export conditions) stay, because that split is about module graphs, not registration. |
| `PluginServices` handed in whole; a plugin re-declares `AppFurniture` structurally | **Overturn** | Services on `ctx`, declared by merging, requested by `inject`. A plugin sees what it names. |
| `probe()` answers both halves at once | **Reframe** | The invariant survives as a property of one waterfall dispatch; the hand-written contract goes. |
| `wake` must be declared on the server door beside `probe` so `chat.scope` can refuse a plugin without one | **Reframe** | `ctx.wakes.register(...)`; `chat.scope` reads the registry. Same check, no field on a manifest. |
| The fence: no general package imports or spells a plugin (`fence.test.ts`) | **Keep** | dsh has the same physics ("no privileged core to patch"). Every claim survives except "no plugin imports a plugin", which becomes "a plugin imports the framework and the service definitions it names". |
| A plugin never imports `@olai/plugin-api` (cycle) | **Overturn** | The cycle existed because `plugin-api` was the registry. As service definitions it is what plugins import, the position `@kolu/surface` already holds. |
| `<plugin>-<kind>` prefixing; the vault's row beats the plugin's claim; the licence consult answers per drawn value | **Keep** | Vault judgement above Cordis's line; the paper "prescribes no concrete scenario" (§5). The composition happens in `ctx.kinds` from `ctx.fiber.name`, which is the registry binding, never the caller. |
| Failure prose is whole sentences owned by the plugin | **Keep** | Nothing in Cordis argues either way; the argument stands. |
| The doorbell cannot read; scope is manual per conversation | **Keep the first, overturn the second** | Write-only is a capability decision and stays. Manual scope was already superseded by node agents (scope is the subtree); under Cordis it is interception metadata on the session's context, derived, never a setting. |
| Disabled means absent | **Keep, strengthened** | Now true at every moment, by construction. |
| The browser receives answers, not rules (#395) | **Keep** | The server still answers the word; the browser Cordis app boots from the roster cell. |
| `@kolu/surface` is the house, not an appliance; never wrap it | **Reframe** | It stays the wire protocol. The house (what is mounted, when, with what) becomes Cordis. Whether siblings can change live is the spike's first question and a possible upstream ask. |
| No `plugin-orchestrator`; the driver is `olai orchestrate`, a wire client with handles minted at the root (eighth sitting) | **Overturn** | The argument was "population one, nothing to probe, the practice is vault data". The practice stays vault data. But a driver that listens to `vault/revision`, holds gate policy, and dispatches lanes is a *group of plugins* in an `orchestrate` profile, the way dsh's agent loop, compaction and plan mode are plugins. That is more testable than a separate process speaking the wire, and it is the only shape in which the orchestrator can later be a node agent (phase 5 of node agents). |
| The receptacle stays home; `packages/plugins` never goes upstream | **Reframe** | Nothing of olai's goes upstream. Cordis comes *down*, as an npins pin hydrated like `@kolu/*`, never as a copied `vendor/` tree (§5). |
| One plugin per agent, never a roster plugin | **Keep** | Each is a fiber; `--plugins` per agent falls out; the paper's cadence argument (independent release clocks) is exactly why they are separate components. |
| kolu and odu land as one PR; property kinds solved now | **Keep as history** | Done and merged; this proposal builds on that tree. |

## 8. Questions the spike answered

Answers from [juspay/olai#472](https://github.com/juspay/olai/pull/472), each backed by a test in `packages/cordis-spike/src/*.test.ts` and re-verified independently (23 pass, fence 30 pass, typecheck clean, `cordis-deps` agrees).

- **Can `@kolu/surface` add and drop a sibling on a live group, and does `connectSurfaces` tolerate a roster change?** No, on both sides: the sibling map is baked at the call, server and client. Re-compose works as a full re-call; a roster change on the client is a reconnect. Framework ask filed first in phase 2 (§5).
- **Does the Proxy `ctx` cooperate with Effect, and does HMR work under Bun?** Proxy and Effect: yes at the access layer (`Effect.gen`, nested `runPromise`, disposers still run), which is what the overlay needs and not a shared lifecycle. HMR under Bun: no (§5).
- **Which registrations are revertible and which are emissions?** Revertible, proved by dispose restoring absence: `ctx.surfaces.register`, `ctx.kinds.register`, `ctx.on` listeners, and a loader row (`disabled: true` disposes the live fiber; a row is never `PENDING` with a half-installed surface). Emissions the runtime does not and must not revert: vault writes, doorbell deliveries, socket dials. No new compensation story is needed for phase 2; kinds validate as text when the plugin is off and a delivered sentence stays in the transcript.
- **Where is intercept metadata for the subtree fence consulted?** On a `vault` service wrapping the store, through `Service.resolveConfig` on the derived context, merged at use with no reload. Not in the store's write path: `store.ts`'s `commit` is Effect and does not read a Cordis context, and putting the fence there would be the store learning Cordis.
- **How does the browser Cordis app boot?** From the roster cell (`@olai/surface`'s `plugins`), now republished on every register/dispose. A served config filtered to the wire and browser doors would be a second roster in a second grammar. The three doors stay. Until the framework ask lands, the browser reconnects when the cell moves.

## 9. Next gen (optional): olai as a Cordis application

*Tagged optional on the human's word, 2026-09-02. Phases 1 to 9 make the plugin **system** a runtime and leave the composition root as the host the plugins hang off. This section is what it would take to go where the paper and dsh go: no privileged core, every part a row. It is not on the plan; it is here so the choice is a reading rather than a rediscovery.*

**What the plan above does not do.** After phase 9, `runtime.ts`'s `bind()` and `serve.ts` are still one fixed function that composes the store, the documents and outlines surface, the chat, git, identity, index, ops, the MCP face and the websocket listener, and plugins are what it mounts. That root is not a component, so the paper's guarantees (recovery, ordering, confluence) stop at its edge: they hold for what is mounted and say nothing about the thing doing the mounting. dsh has no such edge; its session log, tool registry, model adapter and agent loop are rows, and `--dump-config` prints the product.

**What "everything is a plugin" means here, and what it does not.** Not atomization: dsh's core packages are large, and §6.5 of the paper prices finer grain in config and naming. It means three things.

1. **Every part of the root is a `Service` mounted from a row**, with `inject` and `provide`: `vault` (store and revisions), `kinds`, `surface-root`, `documents`, `outlines`, `chat` (sessions, deliveries, the session log), `agents` (engines), `git`, `identity`, `index`, `ops`, `listener`, `mcp`, `web-app`. `bind()`'s one closure, which today captures the store, the chat and every cell together, becomes the wiring those services declare. A part that unloads unwinds its registrations; a part that fails lands `FAILED` alone.
2. **Profiles as bundles.** `olai web`, `olai surface` (headless MCP), `olai orchestrate` and a test-minimal profile are row lists stacking one base bundle. "The empty roster composes" stops being a defended special case: any subset composes, and a test mounts what it needs instead of `bind()` with nulls.
3. **Core's own surface members become contributions to the root.** The rooted bundle gives *siblings* a prefix; core's tags stay unprefixed and byte-identical. So this needs a root-level mount on `@kolu/surface` (unprefixed members merged into the root spec, collisions refused) beside the sibling `mount` kolu#2223 added. Until it exists the surface root stays one plugin, which is fine: replaceability from config is the property, not granularity.

**The enabling piece: an Effect bridge.** Most of the root is Effect (`Scope`, `SubscriptionRef`, `Effect.gen`). One small service holds the Effect runtime, and one helper runs a scoped Effect and returns a disposer that closes the scope. With that, every Effect-managed resource is a revertible effect and core code is mounted rather than rewritten. Build this first; it is what makes the rest incremental.

**What olai would not copy from dsh.** dsh's turn events (`agent/pre-step`, the `tools/*` waterfalls) exist because dsh owns the model loop. olai seats ACP engines and does not. The events olai would own are around session lifecycle: a durable session log with "model-visible means logged" asserted at runtime, a `chat/deliver` waterfall, a prompt-channel seam, and engines as providers on `ctx.agents`. A smaller event surface, and the right one.

**Which parts of the core to promote, and which to leave.** The paper gives four reasons a part earns a fiber, and a part with none of them is data or grammar and should stay what it is. (1) It *acquires* something with a lifetime: a socket, a subscription, a watcher, a child process, so revertible effects replace a hand-written teardown. (2) It is a *seam* with more than one plausible provider, so a row can swap it and the paper's exclusive-binding pattern applies. (3) It is *policy* that composes: several listeners on one event, each replaceable alone. (4) It is instantiated *per scope*: per node agent, per session, per tab. Measured against olai's core (line counts are non-test source, comments included):

| Part of the core (today) | Lines | Reason | Would become | What it enables |
| --- | --- | --- | --- | --- |
| `chat` sessions (`agent.ts`, `chat.ts`: one session in one closure) | ~5,600 of 13,900 | 1, 3, 4 | `ctx.chat` service definition; a session is a fiber under a node-agent scope; delivery policy (`hold mid-turn`, `coalesce`, `queue when absent`) as listeners on a `chat/deliver` waterfall; the wake-scope resolution (manual picker vs derived subtree) as two providers of one seam | Node-agent concurrency without rewriting `agent.ts`; the doorbell rules readable and testable one at a time |
| Engine legs (`chat/src/agents/{claude,opencode,pi}`) | ~1,500 | 2 | one plugin per engine on `ctx.agents` | Already phase 5 |
| `git` (the ledger) | 1,073 | 1, 2 | `ctx.ledger` definition with `git` as its provider | `GIT_OFF` becomes "no provider mounted"; a second backend is a row, not a mode |
| The listener (websocket + `/mcp`) and the MCP face (`server/src/listener.ts`, `mcp/`) | ~2,000 | 2, 3 | transport providers (`ws`, `mcp`, none) and an exposure-policy plugin per face | `olai web` and `olai surface` become profiles that differ by which transport rows they carry, instead of one root with nulls |
| `index` (search) | 891 | 2 | `ctx.search` definition with the core matcher as provider | The parked semantic recall (#165) returns as a second provider row, with no core PR and no ruling to reopen |
| `identity` | 429 | 2 | `ctx.identity` definition; tailscale as one provider | The sitting's own "probe-shaped someday", without waiting for the someday |
| `state` (machine-local state) and `held` | ~500 | 1, 4 | `ctx.state` service keyed by fiber, which `ctx.held` already is | Every plugin's machine-local record on one door |
| The tab's shell (sidebar, outline view, chat panel, preferences, the agents section, the day page) | most of `web` | 3, 4 | client plugins registering into the same slots the tenants use, the way dsh's `ui-session` and `ui-conversation` are | Core's own dressings (date, path, doc, ref) register like a plugin's, so an agent-written dressing (phase 9) is the same kind of thing as a shipped one |
| The orchestrator drivers (wake listener, gate policy, dispatch tools, still unbuilt) | 0 | 1, 3 | a driver group mounted only in the `orchestrate` profile | Phase 7, with nowhere to hide a special case |
| `vault` (the store's revision publisher) | part of `store` | 1 | `ctx.vault`, which exists; the store itself stays a single provider mounted from a row | The test-minimal profile: vault plus kinds and nothing else |

**And what to leave alone.** `format` (33,580 lines) is the vault's grammar: kinds, validation, the meaning consult. It has no lifetime, no second provider, no policy and no scope; it is the language every plugin speaks, and a language is not a component. `ops` is the same one floor up: the operations on a vault are the vocabulary of writes. `surface` is the wire contract. `acp` is the protocol. `fonts`, `child`, `sigterm`, `log` are utilities. Making any of these a fiber would be granularity for its own sake, the cost §6.5 of the paper prices and dsh avoids: dsh's `session`, `tools` and `agent-loop` are large packages, each one row.

**The profiles this would yield.** `web` = vault, kinds, chat, agents, ledger-git, identity, search, listener-ws, mcp-face, web-app, plus the tenants. `surface` = vault, kinds, mcp-face. `orchestrate` = `web` plus the driver group. `test-minimal` = vault, kinds. Each is a row list; `olai --dump-config --profile surface` prints it; and "the empty roster composes" stops being a defended special case because every profile is a subset.
**Re-sequenced plan, if taken.** Phases 2 to 4 as landed (phase 4 is where the Effect facade is built, so the root reuses it), then: **the root's services**, one service at a time behind `runtime.test.ts` and the e2e suite, `chat` and `vault` first because node agents want them; **profiles as bundles** (`web`, `surface`, `orchestrate`, test-minimal) and `olai --dump-config`; then agents as plugins, node agents as scopes, the app shell itself as client plugins (the way dsh's `ui-session` and `ui-conversation` are; the plugin slots themselves already landed in phase 2), the loader surface, the orchestrator profile (now one bundle, not a special case), dynamic plugins. Node agents and the orchestrator profile both presume the root is rows, which is why this track sits before them rather than after phase 8.

**Cost.** `runtime.ts` is the largest single refactor olai would have attempted. The §6.5 integration-component tax lands wherever chat, store and git interleave in one closure today, and each such knot becomes a small integration plugin. One framework ask (the root-level mount) and one internal seam (the Effect bridge) precede any of it.

**Triggers that would make this worth taking.** A second serve profile that genuinely differs from `web` (the orchestrator is the candidate); a third party wanting to replace a core part (the ledger backend, the listener, the index) rather than add beside it; or node agents forcing `chat` and `vault` into services anyway, at which point the first step of this track has already been taken and the question is only whether to keep going.
