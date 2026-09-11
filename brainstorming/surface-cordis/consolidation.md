# Build an app from plugins

**`surface-cordis-app` should run the whole application. Its shell is a plugin.**

Design proposal. API sketches describe the intended public interface; they are not released APIs.

## 1. Your app is a bundle and two entry points

```ts
// server.ts — proposed public assembly API
import { serveBundle } from '@kolu/surface-cordis-app/server'

await Effect.runPromise(Effect.scoped(serveBundle({
  bundle: new URL('./app.yml', import.meta.url),
  resolve: spec => import(spec),
  profile: 'web',
  root: { surface: appSurface, deps: appDeps, faces: appFaces },
  composition: appCompositionContract, // inert cell/schema selection; no live globals
  inputs: appInputs, // explicit app-owned boot services and access policy
})))
```

```ts
// browser.ts — proposed public assembly API
import { followBundle } from '@kolu/surface-cordis-app/browser'
import { browserModules } from './generated/bundle'

const tab = await followBundle({
  modules: browserModules,
  core: { surface: appSurface, name: 'Job board' },
  composition: appCompositionContract, // same inert contract selection
  mount: document.getElementById('app')!,
  retired: showServerReplaced,
})
// At embedding-owner shutdown: await tab.dispose()
// Joins in-flight loading/reconciliation; late completion mounts nothing.
```

```yaml
# app.yml — schematic bundle declaration.
# Generate browser imports from this file, never maintain a second list.
- id: jobs
  name: my-jobs/server
  browser: my-jobs/browser
- id: worker
  name: my-worker/server
- id: renderer
  name: my-solid-renderer/browser
  profiles: [web]
- id: board
  name: my-board-shell/browser
  profiles: [web]
- id: ws
  name: surface-cordis-ws/server
  profiles: [web]
- id: web
  name: surface-cordis-web/server
  profiles: [web]
- id: mcp
  name: surface-cordis-mcp/server
  profiles: [web, headless]
```

The host derives contribution rank once from canonical generated bundle order and supplies it at attachment; consumers do not independently sort by activation time.

The package names in YAML are placeholders. Transport rows own protocol configuration. The host owns the shared address: require an explicit address when active contributions need a listener, and reject an unused address when none do. Headless can still serve MCP or a Unix socket. A transport-free test is a separate profile.

## 2. Plugin contracts and dependencies

```ts
// jobs/server.ts
import { definePlugin } from '@kolu/effect-cordis'
import { Surfaces } from '@kolu/surface-cordis/server'
import { surface, faces } from './wire'

export default definePlugin({
  name: 'jobs',
  needs: [Surfaces, JobStore],
  config: JobConfig, // static Effect schema, decoded before acquisition
  configUpdates: 'reapply',
  apply: config => Effect.gen(function* () {
    const store = yield* JobStore
    yield* (yield* Surfaces).register({
      surface, faces,
      deps: yield* makeJobsDeps(store, config),
    })
  }),
})
```

`surface` uses ordinary `defineSurface`; `makeJobsDeps` and `JobStore` belong to the app. Registration carries the plugin's owner and lifetime automatically. `Wired` supplies an owner-bound sibling client; today its dynamic value is `unknown`, narrowed at the plugin boundary. Runtime contract validation is separate from that TypeScript narrowing. Missing providers keep a component pending; withdrawal disposes its dependents. Independent components may declare different `needs`; their row report still folds component status. Use them for optional components, not for a provider that may never exist: that case needs an explicitly provided narrow broker, or the row can remain waiting and its browser half never load.

```tsx
// jobs/browser.tsx — the board's contract is not a framework import
import { definePlugin } from '@kolu/effect-cordis'
import { Wired } from '@kolu/surface-cordis/browser'
import { board } from 'my-board-shell/contract'

export default definePlugin({
  name: 'jobs',
  needs: [Wired, board],
  apply: Effect.gen(function* () {
    const wire = yield* Wired
    yield* (yield* board).addCard(() => <Jobs wire={wire} />)
  }),
})
```

```ts
// my-board-shell/contract.ts — shell-owned vocabulary
export const board = serviceTag<{
  addCard(view: () => JSX.Element): Effect.Effect<void, never, Scope.Scope>
}>('board.cards')
```

The board plugin provides this service through owner-bound `Offers`, and uses an optional renderer plugin to draw its cards. Another shell can expose documents, a canvas, or no UI at all. Surface never imports `board`, `addCard`, panel geometry or layout slots. Rendering and asynchronous view work close with their contribution scopes. Navigation owns addresses/history and its outlet; layout arranges services and content supplies pages. These are downstream responsibilities, not a required plugin-per-view scheme. A renderer root lasts only as long as its renderer; its contributed layout and child locations can leave sooner.

### Configuration is a lifecycle input

```ts
// Static plugin declaration:
config: WorkerConfig,
configUpdates: 'reapply', // changed config closes the activation and starts another
apply: config => startWorker(config),

// A plugin that consumes changing policy without restarting:
config: WatchConfig,
configUpdates: 'live',
needs: [WatchPolicy], // app-owned, narrow service with revision subscription
apply: Effect.gen(function* () {
  const policy = yield* WatchPolicy
  yield* followWatchPolicy(policy) // owns subscription and update behavior
}),
```

Bundle rows declare available modules, profiles and default enablement. Schemas declare configuration; an app-selected settings provider supplies desired values and enablement. The loader reconciles patches without writing its build declaration. Identical decoded policy preserves activation. A `live` declaration keeps activation config unchanged: the plugin must consume revisions through its declared service, while the host still applies enablement.

The host owns a serialized configuration worker. It observes provider identity as well as revision, rejects publications from withdrawn providers, and finishes already accepted patches independently of the publishing plugin's lifetime. Provider disappearance marks the reading unavailable without reverting applied options. Configuration revisions are distinct from composition revisions and transport epochs.

```text
an authorized durable edit:
  validate → write through the app's persistence service
  → observe that provider's resulting revision → reconcile → acknowledge
```

Writing and runtime application are separate outcomes. If the provider leaves before settlement, report that the write persisted but reconciliation was not confirmed. Configuration and enablement have separate authorization. Durable/session-only control policy belongs to the app; the low-level loader never decides which edits persist. Bootstrap-provider controls must leave a recovery path so stored policy cannot disable its own reader.

Storage format, settings-file discovery, recovery rules and settings UI remain plugins. Schema-derived controls can consume static declarations and redacted reports; they do not belong in the host. Keep authored values, effective values, provenance and validation problems distinct. A broken or unavailable settings source must not silently become an editable empty configuration. Secrets and machine resources have separate environment declarations; secret values never enter reports or browser state. Browser interaction state remains owned by its activation rather than being persisted as server configuration.

## 3. Packages and ownership

```text
your application
  └─ surface-cordis-app       complete startup/build/shutdown recipe
       ├─ surface-cordis     owned surfaces + composition + browser bindings
       │    ├─ Surface      existing schemas, mounts, links, MCP/CLI adapters
       │    └─ effect-cordis existing dependency/lifetime engine bridge
       └─ bundle-selected plugins
            ├─ transports: WS / web assets / MCP / socket
            ├─ optional renderer
            ├─ shell
            └─ domain capabilities
```

`surface-cordis-app` has separate server/browser entry points. Optional adapters depend on the host's service contracts; core host code does not import the app's plugin list.

| Framework | Application plugins |
|---|---|
| Effect/Cordis bridge and its assumption tests | Vault, conversations, agents, document formats |
| Owned Surface registration and incremental composition | Writer identity, ticket minting, credentials |
| Bundle loader/build integration and browser reconciliation | App selection policy, plugin approval policy |
| Shared listener and scoped route/upgrade registration | Shell, navigation, panels, inspector UI |
| Ordered startup, configuration reconciliation and joined shutdown | Settings storage/UI, persistence schemas and migrations |

Split generic listener ownership from connection authorization. Generic write-tag/caller-context plumbing is a candidate framework capability; writer identity and authorization policy belong to the application. Policy is an explicit supplied service, not a silent no-policy fallback. Low-level Surface remains usable without Cordis; the Cordis bridge remains usable without Surface. Preserve Olai's import-closure tests so browser doors cannot pull in Node-only code.

Read-only host observations are declared `HostReports` (server) and `HostTab` (browser) services; the app's composition publication and inspector consume those rather than importing private runtime tables. `HostControl` is separate. The `composition` argument above selects an inert contract; it is not a service locator or captured live singleton. Scoped closures remain valid implementations. Each `followBundle` call owns its full client/reconciliation/report state and returns a disposable instance.

```ts
// Schematic inspector plugin: reading status grants no control capability.
needs: [HostTab, board]
// apply acquires both through Effect and contributes its scoped view.

// Authorized app procedure delegates to the host-owned operation:
set: ({ input }) => control.flip(input.name, input.enabled)
// HostControl.flip forks into hostScope; the request only joins it.
// Both configuration edits and enablement transitions outlive their request.
// Their authorization and persistence policy are separate app decisions.
```

### Ownership is smaller than a plugin

```ts
// Schematic usage of existing ownership patterns, not new released helpers.
const appA = openApp(...) // allocates its own registries
const appB = openApp(...) // cannot overwrite appA's providers

// In a pane's scope, using the actual service acquired through `needs`:
yield* observers.register(...) // pane close withdraws this registration
// Another pane and the plugin itself can continue running.
```

Extract factories, not shared live tables. Each independent consumer owns its holder; release checks a fresh installation token, even if a replacement holds the same service object. Exclusive multi-key claims validate and install atomically; refusal installs nothing. Buses explicitly allow multiple contributors.

Static imports can name contracts and pure helpers. Live values come through declared services; importing mutable state behind a separate readiness signal is still a hidden dependency. Optional providers either register into their consumer or sit behind a narrow broker with explicit absence behavior. A general service locator does not qualify.

## 4. One composition snapshot, several distinct facts

```ts
// Proposed internal observation contract. Not a new plugin authoring obligation.
type Composition = {
  revision: number
  mounts: ReadonlyMap<Owner, {
    activation: ActivationId
    contract: ContractIdentity
    surface: SurfaceDefinition
    faces: Readonly<Record<FaceName, FaceExposure>>
  }>
}

// Separate records, separate lifetimes:
builtModules                 // available static declarations, including disabled rows
desiredConfiguration         // provider identity/revision, values and enablement
selectedModules              // reconciled bundle/profile/policy selection
serverPluginActivations      // server dependency status and failures
composition                  // what is actually mounted and exposed
browserPluginActivations     // status in THIS tab
connectionEpoch              // existing transport identity
adapterAppliedRevision       // acknowledged only after reconciliation succeeds
```

Revision/acknowledgement machinery is conditional: a same-name replacement or reconciliation test must establish that existing mechanisms cannot express the required behavior.

Publish coherent mounted definitions and exposures together. A replacement with the same member names still has a new activation. MCP consumes the snapshot through **existing `reroster`**. Browser reconciliation uses existing link/redial machinery, including Olai's forced socket refresh when upgrade-header policy changes but the surface map does not. An adapter failure stays visible; it never advances a successful acknowledgement. Tabs need not apply the same revision simultaneously. Preserve surviving `Wired` consumers: connection replacement interrupts subscriptions, which resubscribe from a fresh snapshot after a pending interval. Check fresh data, not just connection health. No uninterrupted event delivery is promised, and no blanket browser-plugin restart or duplicate cache is justified by that interval.

Do not put this whole record into core Surface. First prove the contract in the Cordis integration. A later reserved `system/roster` may carry a minimal transport-neutral mounted-surface manifest if another consumer needs it. Bundle metadata, browser chunk policy and per-tab status do not belong in that generic manifest. Plugin enable/disable remains an explicitly authorized control capability; it is not an automatically exposed `system.roster.set`.

Independently built contracts must be checked before binding. Begin with an explicit exact-compatibility rule and visible refusal. Start from existing mount `identity.contractVersion` and `isContractVersionCompatible`; separately prove whether those express the required independently built contract check before adding a new fingerprint. Do not promise schema migrations or an arbitrary third-party plugin marketplace.

## 5. Lifecycle

```text
startup:
  open host → provide inputs → mount settings bootstrap providers
  → accept initial policy → patch remaining bundle rows → settle providers
  → compose surfaces → watch runtime fault → provide listener service
  → settle transports
  → validate policy → open requested listeners

stop one activation:
  close admissions → start interrupting owned calls
  → revoke offers and join dependents → join calls → release resources

stop app:
  mark stopping → close network admission → withdraw listener provision
  → stop/join configuration worker → close host/drain plugin rows
  → close composed surfaces
```

The app recipe owns startup barriers, shared-port ownership and browser composition. When using a settings provider, its bootstrap set and dependencies are app-declared; initial policy is applied before other rows can acquire resources. A settings-free profile uses its explicit build/profile selection. Stop the configuration worker before withdrawing its providers, preventing teardown from scheduling fresh patches. Preserve Olai's lifecycle/gate tests, including dependent cleanup awaiting an interrupted provider call. A hanging finalizer is still hanging; cancellation cannot preempt synchronous JavaScript, and disposal does not undo writes already made.

**Isolation has a real limit today:** plugin initialization and ordinary request failures can be contained, but a mounted Surface connector/install fault or a sibling teardown fault rejects the rooted runtime's `done` and is fatal to the whole bundle. This does not include every periodic read failure; Surface has cell-local error paths too. Preserve fatal shutdown for structural faults. Stronger per-sibling structural isolation is a separate Surface design/proof, not a guarantee this extraction can claim.

Composition-change notifications into plugin effects require owned, gated dispatch. Recomposition remains synchronous; listeners may themselves trigger disposal or recomposition. The dispatch design must preserve that ordering and prove self-disposal cannot deadlock before adopting an awaited `broadcast`.

Within that protocol, register each invocation before it can execute, interrupt that invocation rather than its publisher, and join full fiber exit including children. Child-scope withdrawal and activation stop join the same cleanup; waterfall continuation remains dispatcher-owned. Preserve self-disposal and exceptional revocation paths. An ordinary child scope does not inherit all activation-stage ordering guarantees.

```ts
// Existing Effect pattern: late completion still has a registered release.
const adapter = yield* Effect.acquireRelease(
  Effect.promise(() => openAdapter()),
  adapter => Effect.promise(() => adapter.close()),
)
// Cancellation during acquisition waits for it, then releases the result.
```

Acquire asynchronous resources with release secured across the handoff. Interrupting a promise waiter alone does not dispose its eventual result. Dynamic UI loading must register cleanup before awaiting, recheck ownership before allocation, and dispose partial construction. Shutdown must also refuse or join resource acquisition started after boot.

Durable recovery remains application policy: a job's remote write may have committed before its acknowledgement was lost. Record/reconcile the outcome; never interpret plugin removal as erasing the user's completed work.

The app recipe owns this shutdown sequence. Any shared daemon lifetime abstraction must preserve the same ownership boundary and ordering.

## 6. Prove it with a small app

```text
Counter: move Olai's existing non-notebook fixture across the package boundary.
  web profile          → renderer + tiny shell + counter
  headless profile     → socket/MCP + counter
  in-process profile   → counter, no transport

Job board: prove that the extracted framework handles real lifetime changes.
  disable worker       → detached.held job stops; finalizer records interrupted
  disable board        → UI disappears; jobs remain reachable over MCP
  replace board        → table shell uses the same jobs Surface
  replace jobs         → regression: old handles cannot reach replacement state
  break reconciliation → show failure; do not report the revision applied
  fault a connector    → assert fatal bundle shutdown, not false isolation
  edit worker config   → changed value reapplies; identical value preserves activation
  edit live policy     → subscriber updates without replacing its activation
  disable at boot      → the disabled worker never acquires resources
  withdraw settings    → applied options remain; pending settlement reports its outcome
```

Additional ownership proofs: two hosts in one process; close one of two panes while the other keeps updating; stale release after same-object replacement; conflicting multi-key registration; stop during delayed resource acquisition; optional provider absent then restored; reconnect resumes fresh values; remote write committed but acknowledgement lost. Test outcomes and finalizers rather than requiring a particular helper spelling. Ship parameterized fence checks for both per-door import closure and known live activation state hidden behind exported contracts/re-exports. Local mutable state and owned closures remain valid; syntactic bans are not ownership proofs. Keep the bridge's pinned-engine assumption inventory with the extraction; neither fences nor a passing suite prove arbitrary application cleanup correct.

Counter is the first extraction consumer, not sufficient evidence of generality on its own. The job board adds an independently composed domain and lifecycle tests before finalizing public APIs. Its optional filter component owns activation-local selection and contributes into a child location owned by the board view. Removing the filter leaves the board usable; reconnect preserves selection while replacing the component resets it. Query changes release prior readings, and stale results cannot trigger an action for the new query.

## 7. Delivery and validation

1. Extract the Effect/Cordis bridge with its tests and pinned-engine assumptions.
2. Extract composition, listener ownership and generic configuration reconciliation; run the counter fixture through them.
3. Supply the complete server/browser recipe and adapter plugins.
4. Validate the public interfaces with the job board.

Migrate Olai alongside each slice so its existing tests exercise the shared implementation. Each capability has one owner namespace. MCP uses Surface's rooted catalogs and `reroster`.

Preserve the exact Cordis source pin and required hydration/patch behavior through Kolu's existing npins/pnpm conventions. Verify pin/lock/hash reproducibility and upstream assumption tests before shipping. The bridge owns all private-engine assumptions. Surface changes must meet the repository's paired-consumer and CI requirements.
