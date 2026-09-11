# Surface Cordis — Codex independent proposal

2026-09-08. Written before reading Fable's proposal.

**Extract Olai's working application machinery. Ship a complete app recipe whose renderer, shell, transports, and domain are ordinary plugins.**

Reviewed Olai `ce5f233e1b7591d177f72a5041c1f18eacc8a9fb` and Kolu `56152455be3d7e4e339d406d95eaa2e72152a2db`. Examples below show the proposed public package names and assembly API; they are not released APIs. The Effect plugin shape is already used by Olai.

## What writing an app should look like

```ts
// server.ts — proposed assembly helper
import { runApp } from '@kolu/surface-cordis-app/server'

await runApp({
  bundle: new URL('./app.yml', import.meta.url),
  // Resolution belongs to the consuming package's module graph.
  resolve: name => import(name),
})
```

```yaml
# app.yml — one source for runtime loading and generated browser assets.
# Illustrative rows using Olai's existing bundle/patch approach.
- id: counter
  name: counter/server
  browser: counter/browser
- id: ws
  name: '@kolu/surface-cordis-ws/server'
- id: mcp
  name: '@kolu/surface-cordis-mcp/server'
- id: web
  name: '@kolu/surface-cordis-web/server'
- id: renderer
  name: '@kolu/surface-cordis-solid/browser'
- id: shell
  name: counter-shell/browser
```

The concrete browser-row grammar must come from Olai's generator/loader extraction; this sketch is not a second configuration format. Headless means a patch disabling web/browser rows. The app helper owns startup/shutdown and host services; it does not open a second HTTP listener behind transport plugins. Browser bundles never import server entry points.

## A capability stays an ordinary Surface

```ts
// counter/wire.ts — existing Surface vocabulary
export const surface = defineSurface({
  procedures: {
    counter: {
      read: { input: Schema.Struct({}), output: Schema.Number },
      increment: { input: Schema.Struct({}), output: Schema.Number },
    },
  },
})
export const faces = {
  browser: { 'counter.read': 'tool', 'counter.increment': 'tool' },
  agent: { 'counter.read': 'tool', 'counter.increment': 'tool' },
} as const
```

```ts
// counter/server.ts — same shape as Olai's existing test-counter
import { definePlugin } from '@kolu/effect-cordis'
import { Surfaces } from '@kolu/surface-cordis/server'
import { Effect } from 'effect'
import { surface, faces } from './wire'

export default definePlugin({
  name: 'counter',
  needs: [Surfaces],
  apply: Effect.gen(function* () {
    let count = 0 // belongs to this activation; durable state needs a provider
    yield* (yield* Surfaces).register({ surface, faces, deps: {
      procedures: { counter: {
        read: () => Effect.succeed(count),
        increment: () => Effect.sync(() => ++count),
      } },
    } })
  }),
})
```

Registration is owner-stamped and scope-owned. No global registry, arbitrary owner argument, or parallel `defineServer` callback API. Server effects and browser effects use the same `definePlugin`/`needs` contract. Use the existing independent-component facility when one module has pieces with different prerequisites.

## The shell's vocabulary belongs to the shell

```ts
// counter/browser.tsx — proposed imports, schematic view body
import { definePlugin } from '@kolu/effect-cordis'
import { Wired } from '@kolu/surface-cordis/browser'
import { dashboard } from 'counter-shell/contract'

export default definePlugin({
  name: 'counter',
  needs: [Wired, dashboard],
  apply: Effect.gen(function* () {
    const wire = yield* Wired
    const shell = yield* dashboard
    yield* shell.addCard(() => <CounterView wire={wire} />)
  }),
})
```

```ts
// counter-shell/browser.tsx — schematic shell-owned contract
export default definePlugin({
  name: 'counter-shell',
  needs: [renderer, Offers],
  apply: Effect.gen(function* () {
    const cards = yield* ownedCards()
    yield* (yield* Offers).own('dashboard', cards.service)
    yield* (yield* renderer).mount(() => <Dashboard cards={cards} />)
  }),
})
```

`dashboard`, `addCard`, `ownedCards`, and `<Dashboard>` are shell package code. A canvas shell can expose completely different contracts. The optional Solid renderer owns its DOM root, contributions and error boundaries. Generic `locations()` ownership machinery may remain in the Cordis bridge; neither Surface nor the app host knows sidebar/panel/slot names. Browser calls must be canceled with their view scope; this sketch omits that view implementation rather than pretending a raw promise is owned.

## Package boundaries

| Package (proposed name) | Owns | Starts from |
|---|---|---|
| `@kolu/effect-cordis` | Effect/Cordis lifetimes, declared dependencies, owner-bound services, loader | Olai `packages/effect-cordis`; preserve its tests |
| `@kolu/surface-cordis` | Owned Surface registration, live composition, typed browser wire binding | Olai composition + generic parts of plugin API/wire |
| `@kolu/surface-cordis-app` | Ready-to-run host, bundle resolution/build recipe, startup/shutdown | Olai runtime + bundle + browser bootstrap |
| Optional WS/MCP/web plugins | Admission and adapter lifetime against host services | Olai transport plugins, existing Surface adapters |
| Optional Solid renderer plugin | DOM mount, render ownership and failure presentation | Olai ui-renderer, excluding legacy adapters/app services |
| App/shell plugins | Layout, navigation, documents, persistence policy, credentials | Remain downstream |

These are responsibility boundaries; freeze final package names only after the extraction shows real dependency graphs. Core Surface stays usable without Cordis or Solid. The Cordis bridge stays usable without Surface.

## The part worth adding: one composition value

Olai still coordinates browser and MCP changes using signatures and queues. Kolu already supplies the actual rooted clients and MCP `reroster`. Standardize the input and observation around those mechanisms:

```ts
// Proposed host-internal facts, not extra knobs every plugin must set.
type Composition = {
  revision: number
  mounts: ReadonlyMap<Owner, {
    activation: ActivationId
    contract: ContractIdentity
    surface: SurfaceDefinition
    faces: FaceExposure
  }>
}

// Adapters consume a snapshot and acknowledge successful application.
// MCP delegates to served.reroster(...); it does not get a second catalog engine.
// Browser uses the existing link/reconnect machinery.
```

A new activation with identical procedure names is still new. A new connection is not a new plugin. A selected browser module is not evidence that it activated in every tab. Record these separately; expose adapter applied revision/failure rather than a fictional global `running: true`.

Publish a coherent composition snapshot after validation. Advance an adapter's applied revision only after it succeeds. Validate compatibility before binding an independently built browser contract; initially require exact supported identity and fail explicitly, without promising arbitrary schema migration. Whether existing schema fingerprints suffice needs a focused experiment before inventing another hash.

This is not a distributed transaction: tabs can disconnect, and adapters can fail at different revisions. Old activation handles cannot address replacement state. Do not retry a mutation merely because the connection changed.

## Keep Olai's hard-won lifetime rules

```text
stop activation:
  close admissions
  start interrupting owned calls
  revoke offers and join dependent teardown
  join interrupted calls
  release provider resources
```

Starting interruption before waiting for dependents prevents deadlock when dependent cleanup awaits a provider call. Await full fiber exits, including children. A slow-cleanup warning is not successful disposal. A failure is visible and scoped; it does not mean unrelated plugins must disappear. Bring the existing gate/lifecycle tests upstream with the code.

## Changes to the old HTML plan

| Earlier idea | Decision after current-source review |
|---|---|
| A new shell framework | Keep the ambition; extract the working app recipe. Shell and renderer are plugins. |
| New callback-based plugin DSL | Drop; retain `definePlugin`, `needs`, Effect scopes and owner-bound offers. |
| Root contributions/aliases | Drop. Olai #547 deliberately gives each capability one owner namespace. |
| Build dynamic MCP | Already implemented in Kolu #2234; extract Olai's orchestration around it. |
| Implement lifecycle safety | Already implemented/audited in Olai #554/#557; extract with proofs. |
| Version composition | Still useful; unify activation/composition observations without duplicating transport epochs. |
| Independent plugin compatibility | Keep narrow: explicit validation/refusal, not a new marketplace/update platform. |

## The demonstration

Start with Olai's **existing counter/non-notebook fixture** as the extraction test. Then make a tiny **job board**: a jobs Surface, worker plugin, optional browser board shell, and MCP adapter. Disable the worker while a job is running; disable the shell while MCP continues; swap the shell for a table view. Use a fake slow worker for deterministic lifecycle evidence. No vault, notebook or Olai global service may be necessary.

Ship in slices: bridge extraction with unchanged tests → owned Surface composition → app/bundle recipe and transport plugins → minimal independent app → Olai consumes the same packages. Add composition revision machinery when a same-name replacement/reconciliation test demonstrates what current mechanisms cannot express. Avoid shipping a speculative new stack beside the working one.

## Source anchors

- [Olai bundle/shell/content #538](https://github.com/juspay/olai/pull/538), [transport plugins #536](https://github.com/juspay/olai/pull/536).
- [Shutdown audit #554](https://github.com/juspay/olai/pull/554), [ownership audit #557](https://github.com/juspay/olai/pull/557), [one namespace #547](https://github.com/juspay/olai/pull/547).
- [Kolu rooted MCP/CLI #2234](https://github.com/juspay/kolu/pull/2234).
- Local Olai: `packages/effect-cordis/src/{lifecycle,gate,locations}.ts`; `packages/server/src/{composition,runtime}.ts`; `packages/bundle/{olai.yml,src/bundle.ts}`; `packages/plugins/{test-counter,ui-renderer,layout,ws,mcp}`; `packages/web/src/client/wire.ts`.
- MCP signature/acknowledgement observations are design-review risks, not reproduced bug claims. This proposal is based on current implementation and its tests; no new runtime behavior was implemented or tested here.
