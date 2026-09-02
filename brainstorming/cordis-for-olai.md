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
  ctx.on('vault/revision', snapshot => doorbell.consider(snapshot))         // replaces PluginServer.revision
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

2. **Server on Cordis.** One lane, in this order, each step green alone.
   - **File the framework ask first** (§5): live sibling add/drop on a rooted bundle and `update(surfaces)` on a live `connectSurfaces` connection, or reconnect-per-roster-change as the documented contract. If the ask is slow, phase 2 starts with reconnect-on-change and says so.
   - **Re-compose moves into the composition root** (`packages/server/src/runtime.ts`) before anything else moves. The spike's `Surfaces.recompose` re-implements every surviving sibling; the root's version must compose existing runtimes, set the real `plugins` cell in `@olai/surface` (the spike only shape-proved its roster against `PluginRoster`), and swap the served handlers so a live connection does not keep the old fused group.
   - **`PluginServices` dissolves into services** (`vault`, `env`, `clock`, `log`, `deliveries`, `kinds`, `surfaces`, `wakes`), each a `Service` whose registration methods return disposers attached to the calling fiber; the per-plugin stamp comes from `ctx.fiber.name`.
   - **Hooks become events.** `revision` becomes `vault/revision` (emit); `published` becomes `surfaces/published` (emit); `probe` becomes the `chat/session-start` waterfall (§4.3). **`unloaded()` is not teardown**: it means the store failed to publish, and stays `vault/unloaded`. A half that has teardown beyond its effects gets a distinct `dispose` on the server door; the spike's adapter deliberately calls neither on fiber dispose.
   - **The three rosters become the base bundle**; `--plugins` and the nix option write a `disabled` patch through include's `applyEntryPatches` (the spike's `pluginsPatch` was the same data as a function); `plugin-api` shrinks to service definitions; `fence.test.ts` keeps every claim except plugin-imports-plugin, and the `cordis` arm in its `FRAMEWORK` regex becomes permanent.
   - **Pin hygiene rides along**: `@cordisjs/plugin-include` and `-group` join the pin; the `@ts-nocheck` stamp goes (typecheck the pin under olai's `tsc` or file the delta upstream); the `exports` rewrite covers every imported subpath; `FiberState` is filed upstream as a regular enum or const object and the local numbering is deleted.
   - **Not started in this lane**: HMR (no Bun cache bust exists, §5); intercept in the store (it belongs on the `vault` service, §8); browser slots (phase 5); node-agent scopes (phase 4). Retire `.worktrees/cordis-spike` when the draft closes.
3. **Agents as plugins** (the roster lane, unchanged in intent): `olai-plugin-claude` / `-opencode` / `-pi`, each registering on `ctx.agents`; `AGENTS` in `@olai/surface` becomes the data that registry emits.
4. **Node agents as scopes.** `createScope` per node; sessions are fibers; the subtree write fence is `intercept` metadata on `vault`; reaping is dispose; derived wakes are the session subscribing to `vault/revision` filtered by its subtree. Node-agents phase 1 (roster and switching) is already on master; this phase delivers node-agents phase 2 (concurrency) and phase 4 (derived wakes) as lifecycle rather than as a map in `agent.ts`.
5. **Browser slots.** `ctx.slots` in the SolidJS client replaces the four registries.
6. **Loader surface**: `olai --dump-config`, `olai plugin add`, a preferences page that writes `disabled` onto rows (the nix option writes the same overlay, so both stay honest about which one spoke). HMR in dev only if a Bun cache bust is found (§5); until then a config edit that flips `disabled` is the reload.
7. **The orchestrator as a profile** (see §7): `olai orchestrate` stacks the base bundle with a driver group (wake listener, gate policy, dispatch tools) over the same context, the way dsh's agent loop is a plugin.
8. **Dynamic plugins**: a node agent defines a package into its subtree, a person approves, olai mounts it. dsh's `cordis-host-runner` is the reference.

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
| No `plugin-orchestrator`; the driver is `olai orchestrate`, a wire client with handles minted at the root (eighth sitting) | **Overturn** | The argument was "population one, nothing to probe, the practice is vault data". The practice stays vault data. But a driver that listens to `vault/revision`, holds gate policy, and dispatches lanes is a *group of plugins* in an `orchestrate` profile, the way dsh's agent loop, compaction and plan mode are plugins. That is more testable than a separate process speaking the wire, and it is the only shape in which the orchestrator can later be a node agent (phase 6 of node agents). |
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

*Tagged optional on the human's word, 2026-09-02. Phases 1 to 8 make the plugin **system** a runtime and leave the composition root as the host the plugins hang off. This section is what it would take to go where the paper and dsh go: no privileged core, every part a row. It is not on the plan; it is here so the choice is a reading rather than a rediscovery.*

**What the plan above does not do.** After phase 8, `runtime.ts`'s `bind()` and `serve.ts` are still one fixed function that composes the store, the documents and outlines surface, the chat, git, identity, index, ops, the MCP face and the websocket listener, and plugins are what it mounts. That root is not a component, so the paper's guarantees (recovery, ordering, confluence) stop at its edge: they hold for what is mounted and say nothing about the thing doing the mounting. dsh has no such edge; its session log, tool registry, model adapter and agent loop are rows, and `--dump-config` prints the product.

**What "everything is a plugin" means here, and what it does not.** Not atomization: dsh's core packages are large, and §6.5 of the paper prices finer grain in config and naming. It means three things.

1. **Every part of the root is a `Service` mounted from a row**, with `inject` and `provide`: `vault` (store and revisions), `kinds`, `surface-root`, `documents`, `outlines`, `chat` (sessions, deliveries, the session log), `agents` (engines), `git`, `identity`, `index`, `ops`, `listener`, `mcp`, `web-app`. `bind()`'s one closure, which today captures the store, the chat and every cell together, becomes the wiring those services declare. A part that unloads unwinds its registrations; a part that fails lands `FAILED` alone.
2. **Profiles as bundles.** `olai web`, `olai surface` (headless MCP), `olai orchestrate` and a test-minimal profile are row lists stacking one base bundle. "The empty roster composes" stops being a defended special case: any subset composes, and a test mounts what it needs instead of `bind()` with nulls.
3. **Core's own surface members become contributions to the root.** The rooted bundle gives *siblings* a prefix; core's tags stay unprefixed and byte-identical. So this needs a root-level mount on `@kolu/surface` (unprefixed members merged into the root spec, collisions refused) beside the sibling `mount` kolu#2223 added. Until it exists the surface root stays one plugin, which is fine: replaceability from config is the property, not granularity.

**The enabling piece: an Effect bridge.** Most of the root is Effect (`Scope`, `SubscriptionRef`, `Effect.gen`). One small service holds the Effect runtime, and one helper runs a scoped Effect and returns a disposer that closes the scope. With that, every Effect-managed resource is a revertible effect and core code is mounted rather than rewritten. Build this first; it is what makes the rest incremental.

**What olai would not copy from dsh.** dsh's turn events (`agent/pre-step`, the `tools/*` waterfalls) exist because dsh owns the model loop. olai seats ACP engines and does not. The events olai would own are around session lifecycle: a durable session log with "model-visible means logged" asserted at runtime, a `chat/deliver` waterfall, a prompt-channel seam, and engines as providers on `ctx.agents`. A smaller event surface, and the right one.

**Re-sequenced plan, if taken.** Phase 2 as landed, then: **3. the Effect bridge and the root's services**, one service at a time behind `runtime.test.ts` and the e2e suite, `chat` and `vault` first because node agents want them; **4. profiles as bundles** (`web`, `surface`, `orchestrate`, test-minimal) and `olai --dump-config`; then agents as plugins, node agents as scopes, browser slots (with the app shell itself as client plugins, the way dsh's `ui-session` and `ui-conversation` are), the loader surface, the orchestrator profile (now one bundle, not a special case), dynamic plugins. Node agents and the orchestrator profile both presume the root is rows, which is why this track sits before them rather than after phase 8.

**Cost.** `runtime.ts` is the largest single refactor olai would have attempted. The §6.5 integration-component tax lands wherever chat, store and git interleave in one closure today, and each such knot becomes a small integration plugin. One framework ask (the root-level mount) and one internal seam (the Effect bridge) precede any of it.

**Triggers that would make this worth taking.** A second serve profile that genuinely differs from `web` (the orchestrator is the candidate); a third party wanting to replace a core part (the ledger backend, the listener, the index) rather than add beside it; or node agents forcing `chat` and `vault` into services anyway, at which point the first step of this track has already been taken and the question is only whether to keep going.
