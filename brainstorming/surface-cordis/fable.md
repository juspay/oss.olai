# surface-cordis — an application host over Cordis, Effect and Surface

Independent proposal. Token `SURFACE-CORDIS-FABLE-20260908`. Written before reading any other surface-cordis proposal.

## 0. Reviewed revisions

| tree | revision | what I read |
| --- | --- | --- |
| `juspay/olai` | `ce5f233e1` (#564, HEAD of master, 2026-09-08) | `packages/effect-cordis`, `plugin-api`, `bundle`, `server`, `web`, `surface`, plugins `test-counter`, `ws`, `mcp`, `ui-renderer`, `layout`; `docs/architecture.md`, `docs/internal/plugin-system.md`, `docs/dynamic-plugins.md`, `docs/slot-ownership.md` |
| `juspay/kolu` | `56152455b` (#2234 "surface-mcp/cli: serve a rooted bundle, and follow its roster") — the exact revision olai pins in `npins/sources.json` | `packages/surface`, `surface-app`, `surface-mcp`, `surface-cli`, `surface-remote` as olai consumes them; `docs/atlas/.../electricity.mdx`; `.claude/rules/surface.md` |
| `cordiverse/cordis` | `00278924a` (olai's npins pin, `nix/cordis.nix`): `cordis@4.0.0-rc.9`, `@cordisjs/plugin-loader@1.0.0-rc.6`, `plugin-include@1.0.5` | the engine's `context.ts`/`fiber.ts`/`reflect.ts`/loader `tree.ts`, and olai's assumption table (`packages/effect-cordis/README.md:217-233`) |
| arXiv 2608.25512 | "A Programming Paradigm for Spatiotemporal Composability" (Shi, Zhang, Cui) | temporal vs spatial composability, revertible effects + the accumulator, reactive coeffects, the inertial asynchronous model, the loader as a declarative layer |
| `oss.olai` | `e58886f` | `brainstorming/cordis-for-olai.md` §2, §7, §9, §10; `olai-cordis-perfection.md` (14 findings, all open); `briefs/cordis-*.md`; `plugin-system.md` |

Every claim below that says EXISTING names a file at one of those revisions. Everything marked PROPOSED is a sketch: it does not compile and is not meant to be read as if it did. Kolu-internal line numbers were collected by a research pass I directed over the worktree at `56152455b`; olai line numbers I read myself.

## 1. The one-paragraph version

Olai is already a Cordis+Effect+Surface application framework with exactly one application in it. Roughly 1,900 lines of olai are the framework and know no olai noun: the Cordis bridge (`packages/effect-cordis`, 2,926 lines, zero olai nouns by fence), the sibling registry and rooted recompose (`packages/server/src/runtime.ts` 489 lines, `composition.ts` 196), the one-port listener with scoped contributions (`server/src/listener.ts` 164), the boot/shutdown order (`server/src/serve.ts`), the tab that follows a moving roster (`web/src/client/wire.ts` 402, `web/src/host/runtime.ts` 420), and the roster cell (`@olai/surface/core`). The proposal is to move that machinery into kolu as three packages — `@kolu/effect-cordis` (the bridge, moved verbatim), `@kolu/surface-cordis` (the server host) and `@kolu/surface-cordis-app` (the browser host) — plus the transports as reference ROWS, so that a new app is one bundle file, one `serve()` and one `tab()` call, and every panel, layout, navigation and shell is a row it lists or does not. Headless is a profile that names no transport row and no renderer row; olai proves both halves today (`profile: surface` is MCP-only with no ws/web-app row, `test-minimal` has no transport at all and the listener opens no port; `olai surface <row> <verb>` is a CLI face dialling that serve through `@kolu/surface-cli`). Two things are designed new rather than extracted: the served roster as a framework-reserved cell on the root (a candidate kolu already records, population one), and the split of `TransportSurface` into a generic listener door and app-supplied attribution.

### 1.1 Why Cordis under Effect at all — the first-principles question

Effect's `Layer` builds a dependency DAG once; it has no notion of a provider LEAVING while consumers stand, nor of a consumer re-applying when the provider returns. That is the paper's spatial composability — reactive coeffects — and the pin implements it as fiber state `PENDING → ACTIVE → …` driven by `inject` (`cordis/src/fiber.ts:79-86`, `registry.ts:121-122`). The temporal half — every effect carries an inverse the runtime holds and replays in reverse, the "accumulator" (paper §3, "the composite of the inverses of the effects performed so far") — Effect already has as `Scope`, and the bridge says so: "Effect's `Scope` and the paper's accumulator are the same idea, spelled twice; there is one of them here" (`effect-cordis/README.md:39-46`). So the honest division is: **Cordis for the coeffect (waiting/unloading/re-applying) and the declarative loader (rows, patches, runtime flips); Effect for the effect (scope, finalizers, interruption, the `R` channel)**. Olai considered and declined the Effect-only alternative ("reimplementing epochs, PENDING, provider replacement and a declarative entry tree over `Scope` … not taken", `effect-cordis/src/index.ts:33-39`). I re-asked and reach the same answer, with one sharpening: the bridge's value is exactly the parts it withholds — `openHost`, `provide` and the stamp read off the fiber (`plugin-api/src/runtime.ts:27-58`) — because that is what turns "a plugin cannot sign another's registration" from a rule into a shape. A framework that exposed Cordis directly would lose that.

What the pin has that olai deliberately does not use, and this proposal does not either: isolation realms (`ctx.isolate`), interception (`ctx.intercept`), the engine's own event bus (`ctx.on`/`emit`, a bare `Reflect.apply` loop with no `try`), and `fiber.update` HMR (`cordis-for-olai.md` §10). Each is a later phase for an app, not a host concern.

## 2. What an application writes, before and after

### 2.1 Today (EXISTING, olai)

A plugin is two Effects, one row. The server half — `packages/plugins/test-counter/src/server.ts` (15 lines, whole file):

```ts
export default definePlugin({ name, needs: [Surfaces], apply: Effect.gen(function*() {
  let count = 0
  yield* (yield* Surfaces).register({ surface, faces, deps: {
    procedures: { counter: { read: () => Effect.succeed(count), increment: () => Effect.sync(() => ++count) } },
  } })
}) })
```

The browser half — `test-counter/src/browser.tsx:11-14`:

```tsx
export default definePlugin({ name, needs: [Wired, rendererSlots], apply: Effect.gen(function*() {
  const wired = yield* Wired                       // this plugin's own sibling client
  const slots = yield* rendererSlots               // the ui-renderer ROW's contract, not the host's
  yield* slots.contribute(root, () => <main>…</main>)
}) })
```

The row — `packages/bundle/olai.yml:245-250`:

```yaml
- id: test-counter
  name: olai-plugin-test-counter/server
  disabled: true
  profiles: [test-minimal]
```

That is the whole plugin authoring contract and it is already framework-shaped: `needs` is both the Cordis `inject` list and the Effect `R` channel (`effect-cordis/src/plugin.ts:1-20`), each registration is an `acquireRelease` on the plugin's own scope, and the stamp (`name`) is read once off the fiber (`plugin.ts`, "THE STAMP, READ ONCE").

What is NOT framework-shaped is what the application writes around it. `packages/server/src/serve.ts:149-347` is 200 lines whose entire content is ordering: open the host, provide inputs, mount rows under a profile patch, settle, open loading, report, bind the rooted surface, watch the runtime fault, register four finalizers in one exact order, provide `TransportSurface`, settle again, check upgrade headers, start the port. `packages/server/src/runtime.ts:127-489` is `bind`: the sibling registry made into the rooted wire, with `mounted`/`standing`/`leaving` tables, a `moving` suppression flag, gate-at-the-tail, a `rosterMoved` bell for faces that hold resolved tables (MCP), and the `plugins.set` procedure forked into the serve's scope so a flip that closes the caller's own socket still finishes. `packages/web/src/client/wire.ts:49-69, 138-295, 359-383` is the tab's twin: dial the root with `surfaces: {}`, watch the roster cell, load chunks, `redial`, force a socket refresh when the header policy may have moved, `composeTo`, queue a second roster frame behind the first. None of those files contains an olai noun that matters; every second consumer would rewrite all of them and get one of the orderings wrong (olai got several wrong on the way: `runtime.ts:143-165`, `serve.ts:221-251`, `wire.ts:143-157` are each a paragraph about an ordering bug).

### 2.2 Proposed (PROPOSED, does not compile)

A new application is one bundle file and two entry points.

```yaml
# bundle.yml — the whole plugin list, and the only place a plugin is named by hand
- id: counter
  name: myapp-plugin-counter/server         # server half; the loader resolves it at mount
  browser: myapp-plugin-counter/browser     # optional browser half
- id: shell
  name: myapp-plugin-shell                  # browser-only row: claims the renderer root
  profiles: [web]
- id: ws                                    # reference transport rows, shipped by the framework
  name: "@kolu/surface-cordis/rows/ws"
  profiles: [web]
- id: web-app
  name: "@kolu/surface-cordis/rows/web-app"
  profiles: [web]
- id: unix-socket
  name: "@kolu/surface-cordis/rows/unix-socket"
  profiles: [headless]
```

```ts
// serve.ts — the server entry
import { serveBundle } from "@kolu/surface-cordis"
await Effect.runPromise(Effect.scoped(serveBundle({
  bundle: { baseUrl: import.meta.url, path: "./bundle.yml", resolve: (spec) => import(spec) },
  profile: process.argv.includes("--headless") ? "headless" : "web",
  listen: { host: "127.0.0.1", port: 7714 },      // an unused option on a profile with no transport row
  inputs: [MyAppBoot, () => ({ root: process.cwd() })], // app-owned services provided BEFORE any row mounts
})))
```

```ts
// tab.ts — the browser entry
import { followBundle } from "@kolu/surface-cordis-app"
await followBundle({ rows: BROWSER_ROWS, mount: document.getElementById("root")!, retired: () => console.warn("replaced") })
```

The renderer is a row, not the host — `packages/plugins/ui-renderer/src/browser.tsx:19-48` already reads this way: it `needs: [BrowserMount, Offers]`, opens a `locations` registry, offers `slots`, and `render`s `root` into the element the host handed it. A headless tab (no renderer row) dials, follows the roster, mounts server-paired browser halves that draw nothing, and never touches the DOM. A headless SERVER (profile `headless`) mounts content rows and no port: `listener.ts:95-101` already closes the port when no non-passive contribution stands, and `serve.ts:302-305` names that as the design ("passive media routes alone must not turn a headless, transport-free selection into a network server").

### 2.3 What a plugin author sees — unchanged

The four doors a plugin imports today from `@olai/plugin-api` become framework doors plus app doors:

| door | today (EXISTING) | proposed |
| --- | --- | --- |
| `definePlugin`, `serviceTag`, `detached`, `broadcast`, `waterfall`, `registry`, `roster`, `location(s)`, `standing` | `@olai/plugin-api` re-exports `@olai/effect-cordis` verbatim (`plugin-api/src/runtime.ts:95-123`) | `@kolu/effect-cordis` — same list, same withholding of `openHost`/`provide`/`settled`/`namedBy`/`offered` |
| `Surfaces`, `Offers`, `Env`, `Clock`, `LocalState`, `Bundle`, `HostLoading` | `@olai/plugin-api/services` | `@kolu/surface-cordis` (`Surfaces`, `Offers`, `Bundle`, `HostLoading`); `Env`/`Clock`/`LocalState` stay APP doors — a framework has no opinion about a machine's state home |
| `TransportSurface` | `@olai/plugin-api/transport` | `@kolu/surface-cordis/listener` — generic half; olai's attribution rides `services(connection)` (see §4.3) |
| `Wired`, `Offers`, `BrowserMount` | `@olai/plugin-api` (browser door) | `@kolu/surface-cordis-app` |
| `Slots`, `Faces`, `Bar`, `Links`, `Clocks`, `rendererSlots`/`root` | olai's ui-renderer, layout, navigation rows' `/contract` doors | unchanged — they are rows' contracts, never the host's (`docs/slot-ownership.md`) |
| `Vault`, `Deliveries`, `Kinds`, `Wakes`, `Agents`, `Watching`, `Identity`, `Tools`, `Ops`, `Served` | `@olai/plugin-api/services` | unchanged, olai's own; provided by olai's `serve()` through the framework's `inputs`/`provide` seam or offered by olai rows |

The rule that sorts a door: the framework keeps a service iff it is minted from a plugin's stamp AND names only row/surface/service vocabulary. `Surfaces.register` qualifies (`plugin-api/src/services.ts:1540-1551`: `siblings.claim(plugin, {...sibling, name: plugin})`). `Deliveries` does not (it names a conversation).

## 3. Package ownership

Three packages, because three volatilities. One package would repeat the `@kolu/artifact-sdk` "two volatilities under one roof" smell the atlas flags (`electricity.mdx:69-91`).

| package | volatility it hides (test ②) | contents (EXISTING source it is extracted from) | designed new |
| --- | --- | --- | --- |
| `@kolu/effect-cordis` | the Cordis pin — unstable API, 15 behaviour-only assumptions tabled in `packages/effect-cordis/README.md:217-233` | `packages/effect-cordis/src/*` moved verbatim: `plugin.ts` `definePlugin`/`detached`; `host.ts` `openHost`/`closeHost`/`provide`/`mountPlugin`/`settled`/`rowReport`/`hostChanges`/`offered`/`namedBy`; `lifecycle.ts` `offer`/`OfferConflict`; `loader.ts` `mountRows`/`flipRow`/`rowConfigs`; `locations.ts`; `registry.ts`; `broadcast.ts`; `waterfall.ts`; `standing.ts`; `module.ts`; every test | nothing. Its `nix/cordis.nix` pin and `check-hydrated-deps.sh` discipline move with it |
| `@kolu/surface-cordis` (server host) | how a served roster is assembled and moves — an axis that has moved four times in kolu alone (`electricity.mdx:48`) and once more in olai (#546, one tag per member) | `server/src/runtime.ts` `bind` → `composeRoster`; `server/src/composition.ts` `composeCapabilities` (faces + writes over kolu's `implementRootedSurfaces`, `packages/surface/src/server.ts:4134`); `server/src/listener.ts`; `plugin-api/src/services.ts:1472-1681` `openPlugins` minus olai doors; `plugin-api/src/loading.ts` `openLoading`; `plugin-api/src/owned.ts` (the `<plugin>.<word>` grammar); `plugin-api/src/slots.ts` (generic slot vocabulary, "no application catalog"); `plugin-api/src/authority.ts` `authorityAt` (write attribution over declared `writes` tags); `bundle/src/bundle.ts` `mountBundle`/`profilePatch`/`pluginsPatch`/`setRow`/`reportBundle`; `server/src/pluginPolicy.ts` `pluginPin` (the exact/delta/omitted grammar; the flag spelling stays the app's); `serve.ts:149-347` as `serveBundle`; the reference rows `ws`, `web-app`, `unix-socket`, `mcp-endpoint`, `stdio` | `system/roster` reserved cell (§4.1); `Listener` door split (§4.3); `serveBundle` as the one ordered entry (§4.4) |
| `@kolu/surface-cordis-app` (browser host) | how a tab follows a roster that moves under it — redial, chunk load, compose, retry | `web/src/client/wire.ts`; `web/src/host/runtime.ts` (`composeTo`, `recompose`, `attachRenderer`); `web/src/host/{bootstrap,loading,boot-status}.ts`; `plugin-api/src/browser.ts:705-833` `Wired`/`openApp`; `plugin-api/src/mount.ts` `BrowserMount`/`BROWSER_BOOT_PATH`; `bundle/generate.ts` (rows → literal `import()` table, css chain, testid merge) as a build helper | `followBundle` (one call: dial root, watch `system/roster`, redial, compose); the redial-on-roster option the electricity row asks for |

Direction of arrows, all pointing out: `myapp → @kolu/surface-cordis → @kolu/surface{,-app,-mcp,-cli} + @kolu/effect-cordis → cordis, effect`. Rows shipped by the framework import the framework's doors and nothing of an app. The olai fence (`bundle/src/fence.test.ts`: "Cordis is an engine nobody outside one package sees") is kept as-is, with the one package now living in kolu.

What stays in olai, by name: `Vault`/`Ops`/`Directory`/`Deliveries`/`Kinds`/`Wakes`/`Agents`/`Watching`/`Identity`/`LocalState`/`Tools`/`Served` and their provisions; `who.ts`; `gitPolicy.ts`; `localState.ts`; `propKinds.ts`; `pluginPolicy.ts` (`--plugins` grammar is CLI policy, the PATCH mechanism is the framework's); the MCP row's tickets and write reservations; every content and shell row; `docs/plugins/*`.

## 4. The four things that are new

### 4.1 The roster is a framework fact on the root (`system/roster`)

EXISTING: olai publishes the roster as an app cell — `@olai/surface/core`'s `plugins` (`packages/surface/src/core.ts:48-60`, `equals: sameRoster`, `arrayKey: "name"`), republished from `recompose` (`runtime.ts:344`), watched by the tab (`wire.ts:359-383`), and switched by an app procedure `plugins.set` (`core.ts:63-76`). Kolu records this exact shape as a candidate electricity, population one, with the intended home spelled out: "a framework-reserved `system/roster` cell on the ROOT ... plus a `connectSurfaces` option that redials on its change. Home would be `@kolu/surface` beside the reserved `system/*` members — not a new package" (`electricity.mdx:48`).

PROPOSED, in two halves so the `@kolu/surface` change stays minimal and gated (`.claude/rules/surface.md` — a paired drishti PR):

```ts
// @kolu/surface — one reserved cell beside system/live and system/identity  (PROPOSED)
system.roster: Cell<ReadonlyArray<{ key: string; state: "running" | "waiting" | "failed" | "off"
                                     fault?: string; missing?: string[]; browser?: { chunk: string | null } }>>
// arrayKey "key", equals by (key,state,fault,missing,chunk); written ONLY by implementRootedSurfaces'
// owner through a ctx the host holds — never by a sibling, never by a wire client.

// @kolu/surface-app/solid  (PROPOSED)
connectSurfaces({ core, surfaces, retired, followRoster: (rows) => Promise<Record<string, Surface>> })
// when supplied, the seam watches system/roster and calls conn.redial(await followRoster(rows)) itself,
// serialised one-at-a-time exactly as olai's `rerost` queue does (wire.ts:138-163).
```

Why the root rather than a sibling: a sibling can leave, the root cannot, and the liveness probes already aim at the root for that reason (`connectSurfaces.ts:220-225`: the root's bare `system/identity`/`system/live` are the reserved probe target "which is what makes a rooted wire safe while the sibling set varies"). Why a reserved member rather than each app's cell: forgetting it fails silently — clients never learn about a new sibling and nothing errors (`electricity.mdx:48`; the same warning is repeated in code at `packages/surface/src/server.ts:3963-3981`). Kolu already holds the three reserved members in one table, `RESERVED_MEMBERS` (`define.ts:597`), always reachable on a gated face (`define.ts:612-620`); a fourth row there is the whole `@kolu/surface` change.

What olai keeps beside it: its own `plugins` cell shrinks to the olai-only words (`wake`, `pin`, `config`, `section`, `carrying`) or is dropped in favour of a `plugins` SIBLING row that publishes them. `carrying` (which rows would go `waiting` if this one is switched off — the join of `offers()` × `namedBy`, `runtime.ts:78-91`) is generic and moves to the framework's roster row.

The switch (`plugins.set`) becomes a root procedure `system.roster.set({key, enabled})`, implemented by the host exactly as olai does today — forked into the host's scope and joined from the request (`runtime.ts:188-208`), refusing with a typed not-found — because a flip that turns a transport row off closes the very connection asking for it.

### 4.2 The composition loop is the host's, whole

EXISTING and moved as one unit: `bind`'s `recompose` (`runtime.ts:268-351`). The properties it keeps are the ones a second consumer would get wrong, so they are listed here as the contract rather than left in comments:

- arrivals are mounted before departures are dropped; a survivor is touched by neither loop, so it keeps its handler identity, stores, channels and running sources (`runtime.ts:240-267`; kolu's `mount` is transactional and incremental in state, `packages/surface/src/server.ts:3989-4003`, channels namespaced `<key>#<gen>/<name>` per mount, `:4120-4128`).
- a returning key waits for its previous generation's `drop()` to settle, as a continuation, never an await (`runtime.ts:281-289`; kolu's `mount` refuses a key whose previous generation has not settled, `server.ts:4277-4288`). `drop()` lives on the `MountedSurface` value, never as `drop(key)` on the runtime, so only the holder can retract its sibling (`server.ts:3852-3859`); it is idempotent and never rejects, a teardown fault reaches `done` (`:3877-3884`); the moment it is called both faces refuse with `SurfaceSiblingDropped` — wire handlers on open connections AND the sibling's own `ctx` (`:3893-3902`, `:3809`).
- the gate is read off what is SERVED, at the tail, one value per generation — because kolu's `restrictHandlers` compares exposure universe and group as a set equality at every accept and terminates the socket on a mismatch (`composition.ts:62-92`, `runtime.ts:324-341`).
- the roster is not republished while a flip is in flight (`moving`), and a registration that landed during the flip is republished after it (`pendingStatus`) (`runtime.ts:357-396`).
- faces that hold a resolved table are rung after the gate (`rosterMoved` → MCP's `reroster`, `runtime.ts:472-475`, `plugins/mcp/src/endpoint.ts:75-89`).
- compose once, in line, before the change loop is forked, and let `hostChanges`' initial emission close the registration window (`runtime.ts:413-428`).

Nothing in that list names a vault. The host publishes `{ group, handlers, faces, writes, rows, rosterMoved, done, close }` — today's `Bound` (`runtime.ts:20`).

### 4.3 The listener door splits into a generic half and an app half

EXISTING: `TransportSurface` (`plugin-api/src/transport.ts:21-86`) carries both: `register`/`routes`/`live`/`services`/`report`/`allowedOrigins`/`clientDist`/`hostname`/`agentRows`/`agentRosterMoved` (generic) and `who`/`upgradeHeaders`/`token`/`writeReservations`/`browserBoot` (olai's identity, ticket and policy). The ws row (`plugins/ws/src/server.ts:65-95`) reads only the generic half plus `upgradeHeaders()` and `services(connection)`.

PROPOSED: `Listener` (framework) keeps the generic half; the two seams the ws row already uses become the app's ONLY inputs:

```ts
// @kolu/surface-cordis/listener  (PROPOSED — shape is transport.ts:21-86 minus olai's fields)
interface Listener {
  register(contribution: ListenerContribution): Effect<void, never, Scope>   // routes | upgrade | passive
  live(face: string): ServedGeneration                                       // group+handlers+expose, read per accept
  rows(): ReadonlyArray<{ name; surface; resources; tools }>; rosterMoved(run): () => void
  services(connection): Layer<never>            // APP-supplied per-connection context (olai: CurrentWho)
  upgradeHeaders(): ReadonlyArray<string>       // APP-supplied thunk (olai: Identity row's headers)
  allowedOrigins; hostname; report; clientDist
}
// serveBundle({ ..., listener: { services, upgradeHeaders, allowedOrigins, clientDist } })
```

`token`, `writeReservations`, `browserBoot`'s policy and `who` become olai doors olai's own rows read. Face NAMES (`browser`, `agent`) stay data the app passes (`faces` maps per sibling), never words the framework knows — the same domain-blindness line `liveWhen` draws (`electricity.mdx:65-67`).

Why the `ws` row composes kolu's granular seam (`acceptSurfaceSocket` `surface-app/src/server.ts:801` + `serveSurfaceSocket` `:1015`, one `RpcServer` per connection over shared handlers) rather than calling `serveSurfaceApp` (`surface-app/src/serve.ts:613`): the convenience owns a whole HTTP server, its shell routes and its socket population, answers a plain URL and tears down by scope finalizer (`serve.ts:858-891`) — so calling it from a row would either take a second port or make one row the owner of every other transport's lifetime (`plugins/ws/src/server.ts:4-10`). The framework host owns the port; rows contribute. This is the one place the proposal reads against the `surface` skill's "don't hand-roll a listener" and the reason is olai's, not mine.

Reference rows the framework ships, each a plugin whose whole body is what olai's row already is: `ws` (`plugins/ws/src/server.ts`, 97 lines, imports only `@kolu/surface*` and the door), `web-app` (`plugins/web-app/src/server.ts`, 49 lines: static routes, service worker, manifest), `unix-socket` (new: `serveOverUnixSocket` on the same `live()`), `stdio` (new: `serveOverStdio`), `mcp-endpoint` (the generic part of `plugins/mcp/src/endpoint.ts`: route + `serveSurfaceAsMcp` rooted bundle + `reroster` on `rosterMoved`; olai's ticket mint stays an olai row that `Offers.own("ticket-mint")` into a door the endpoint row `needs` optionally — this is the one place the split is not clean, §6).

### 4.4 `serveBundle` is the one ordered entry, and the order is the contract

EXISTING: `serve.ts:120-148` and `221-251` argue the order in prose. PROPOSED: the order is the function, and an app cannot spell it differently.

```
open host → provide app inputs → mount rows (profile patch ∘ pin patch ∘ config patches)
→ settle(built)   [barrier 1: providers stand]
→ compose roster (in line) → watch runtime fault → listener (accumulating, no port)
→ provide Listener door → settle(built)   [barrier 2: transports registered]
→ app pre-bind check (olai: checkUpgradeHeaders) → start port (or none) → publish `system/roster`
shutdown, reverse: stopped-flag → port → rows drain (each dispose recomposes) → composed surface close
```

Two barriers rather than one because content runs headless and transports wait on the door (`serve.ts:1-10`). The four finalizers run in exactly the order `serve.ts:221-251` gives, for the reasons given there (a surface closed before its rows drain retracts ctx from rows still tearing down; a fault observed after `stopped` is an ordinary shutdown). Kolu's lifetime audit names this whole shape as a missing concept — a serving process's **tenure**, "one missing concept expressed 17 times" — and defers a shared shutdown sequence until two real consumers need it (`surface-lifetime-audit.mdx:85-100`). `serveBundle` would be that second consumer's spelling of it; whether it lands in `surface-daemon` or here is a question for the framework owners (§9).

`serveBundle` also observes the composed runtime's `done` as fatal, as `watchFault` does today (`serve.ts:252`, `fault.ts`), because kolu states it as a law: "a `done` rejection is structural wiring death a serving site must treat as fatal — no log-and-keep-serving" (`packages/surface/src/server.ts:2416-2424`).

## 5. Lifecycle guarantees the host keeps (and who keeps them)

| guarantee | kept by | source |
| --- | --- | --- |
| a plugin held `waiting` until every named service exists, unloaded when one leaves, re-applied when it returns; `needs` = `inject` = `R` | bridge | `effect-cordis/src/plugin.ts`, README "definePlugin" |
| every registration is a finalizer on the plugin's own scope; a failed `apply` installs nothing, siblings keep running | bridge | README "A registration carries its own undo" |
| offers revoked and dependents joined BEFORE the provider's own resource finalizers run, including finalizers registered after the offer | bridge (`lifecycle.ts`) | README, two pin couplings named |
| duplicate provider refused with both owners named (`OfferConflict`) | bridge + host (`Offers.own` sentence) | `services.ts:1582-1599` |
| the per-plugin stamp cannot be spelled by a caller; `mountPlugin` and `openHost` withheld from plugins (name forgery) | bridge doors | `plugin-api/src/runtime.ts:27-58` |
| stop mid-start interrupts the Effect fiber and unwinds what was installed (olai's adaptation, not the paper's) | bridge | README "Two things this package does NOT claim" |
| `settled` waits out movement, bounded, decides nothing | bridge | README "one verb that translates nothing" |
| disabled = absent at every moment: no tag, no handler, no expose row, no chunk fetched | host + app host | `bundle/README.md` "What a disabled plugin is"; `rows.ts:40-49` |
| survivors keep identity across recompose; returning key waits for its drain | host | §4.2 |
| one gate per generation, read off what is served | host | §4.2 |
| a browser half is mounted only over a wire that carries its sibling; a failed half is disposed, retained as a report, retried next frame | app host | `web/src/host/runtime.ts:258-385` |
| a page is never drawn from a half-composed table (compose counter) | app host | `host/runtime.ts:41-103` |
| one redial at a time, queued not dropped | app host | `wire.ts:138-163` |
| stale tab / server replaced → `retired`, required option, no default | kolu surface-app (already) | `wire.ts:56-68`; the pid compare at `surface-app/src/server.ts:641-667` |
| a redial keeps `core`, `clients` (same object), `link`, `readout`, `health`; only `dispose` is terminal; standing subscriptions re-open on their own retry fence | kolu surface-app (already) | `connectSurfaces.ts:439-477`, `:527` |
| a sibling's owned-source or teardown fault ends the WHOLE bundle, root included — there is no per-sibling quarantine, retry or crash budget | kolu surface (already, by law) | `packages/surface/src/server.ts:2435`, `:3903-3908` — see §6.9 |

## 6. Unresolved limitations (stated, not solved)

1. **Reconnect per roster change.** An open connection keeps the roster it dialed; Effect RPC bakes the group into the client at `openWireLink`. The browser redials in place (kolu #2228), MCP rerosters in place, a CLI dials per call. The framework ask ("a served wire that follows the roster") is recorded in `services.ts:533-541` and stays open. `system/roster` makes the redial automatic; it does not remove it.
2. **Typed sibling clients are lost to a data roster.** `Wired.client()` is `unknown` and each browser half narrows once at its edge (`wire.ts:242-250`, `test-counter/src/browser.tsx:26`). A framework cannot type a plugin's client without learning its members. This is a cost, accepted.
3. **A third-party plugin still rebuilds the app**, because browser chunks are split on literal `import()` at build time (`rows.ts:19-22`). `generate.ts` makes it one row rather than three lists; it does not make it zero builds. The vault-defined plugin path (`docs/dynamic-plugins.md:80-88`: compile at serve, chunk URL in the roster, version in the path) is the out-of-tree seam and stays app-owned policy (approval); the framework carries only `chunk` on the roster row and the `HostLoading` door (`plugin-api/src/loading.ts`).
4. **Two bundlers.** olai builds with `bun build`; kolu's `@kolu/surface-app/vite` is Vite. `generate.ts` is bundler-neutral (it writes source); the chunk URL map (`BROWSER_MODULES_ID`, `wire.ts:190-194`) is not. The framework ships the generator and leaves the map's producer to the app's build.
5. **The MCP endpoint's ticket policy** is olai's write-authority model; the endpoint row cannot be generic without an optional door for it, and "optional to exist" must be a broker rather than a component (`plugin-system.md:1461-1486`). Proposed: the endpoint row `needs` a closed broker `McpPolicy { ticket(): …; writes(): … }` the app provides, defaulting to no-ticket/no-writes.
6. **Cordis pin instability** stays exactly where it is — one package, one table, four upstream asks (`effect-cordis/README.md:188-240`). Moving the package to kolu moves the table; it does not shrink it.
7. **Cancellation is cooperative**; a synchronous `apply` cannot be preempted (README).
8. **Test ③ has population one.** Every candidate row kolu records is "prove-then-extract". No second application exists yet; §8 names the cheapest honest one and the human should rule on it before stage 2.
9. **Failure isolation stops at `apply`.** The bridge contains a plugin whose `apply` dies (siblings keep running) and a handler that throws (a per-request defect). It does NOT contain a sibling whose running SOURCE dies after mount: kolu treats that as structural wiring death of the whole rooted bundle (`server.ts:2435`, `:3903-3908`), `watchFault` reads it as a fault, and the process goes down. A plugin host cannot promise "a broken plugin cannot take the app down" until the framework has a per-sibling fault channel. This is a framework ask, not something the host can paper over, and the "no fallbacks" rule says it must not try.
10. **Open audit findings travel with the code.** `olai-cordis-perfection.md` lists fourteen findings, none closed. Five are defects of the machinery this proposal moves rather than of olai: #1 (a `handlers.read()` snapshot can dispatch to a disposed plugin), #6 (what browser clients promise across reconnection is unverified), #11 (MCP's non-atomic await-then-`addFinalizer`), #13 (the `OfferConflict` prose match needs a pin test), #14 (the bridge's upstream assumption table is stale on concurrent disposal). Moving the code moves the findings; stage 1 should carry them as its own acceptance list.
11. **Row `components`** (a module exporting independently injected sibling fibers) are olai's convention layered on the bridge, not a pin feature (`cordis` `config/entry.ts:9-16` has no such field). They are kept — the browser host mounts them under `<row>/<local>` (`web/src/host/runtime.ts:298-322`) — but they are a bridge feature the framework owns and documents, not something the loader gives for free.

## 7. One convincing small demo

`packages/surface-cordis/example/counter/` — the olai `test-counter` fixture re-homed, with its own 30-line shell row, run three ways from one bundle:

```
$ bun example/counter/serve.ts --profile web        # ws + web-app + shell + counter: a page with a button
$ bun example/counter/serve.ts --profile headless   # unix-socket + counter: no port, no DOM
$ counter-cli --socket .counter.sock counter increment          # @kolu/surface-cli over unixSocketLink
$ bun test example/counter                          # directLink(router) against serveBundle in-process, no rows of transport
```

What it proves, and only what it proves: (a) one bundle, two profiles, zero code change between them; (b) the shell is a row (`shell/browser.tsx` claims `root` through the renderer row's contract, exactly `test-counter/src/browser.tsx:11-14`); (c) switching the shell row off at runtime leaves the counter's server half serving and its browser half `waiting` on `rendererSlots`, reported on `system/roster` — the state the runtime is for (`ui-renderer/src/index.ts:13-26`). The fixture already exists and already runs "headless and rendered in a tiny shell by ordinary host composition" (`test-counter/src/wire.ts:1-2`); the demo is moving it across the package line.

## 8. Staged proof plan

| stage | move | proof (green means done) | gate |
| --- | --- | --- | --- |
| 0 | `@olai/effect-cordis` → `@kolu/effect-cordis`, verbatim, with `nix/cordis.nix`, the assumption table, `upstream.test.ts` | olai hydrates it like every `@kolu/*` (`check-hydrated-deps.sh`); olai's fence still holds "one package names cordis"; olai CI green with `@olai/effect-cordis` a one-line re-export | none — new package, no surface API change |
| 1 | `@kolu/surface-cordis`: `composeRoster` (= `bind` − olai words), `composeCapabilities`, `listener`, `openPlugins` core, `mountBundle`, `serveBundle`; `system/roster` on the root; `Listener` door | olai's `runtime.test.ts`, `composition.test.ts`, `listener.test.ts`, `shutdown.test.ts`, `headless.test.ts`, `transports.test.ts`, `passive-routes.test.ts` move to kolu and pass; `serve.ts` shrinks to olai doors + `serveBundle`; olai e2e green; audit findings #1, #11, #13, #14 closed in the moved code (§6.10) | **drishti pair PR** (`system/roster` is a `@kolu/surface` contract change) + odu-impact verdict |
| 2 | `@kolu/surface-cordis-app`: `followBundle`, `openApp`, `Wired`, `BrowserMount`, `composeTo`, bootstrap; `connectSurfaces.followRoster` | olai's `host/runtime.test.ts`, `bootstrap.test.ts`, `loading.test.ts` move and pass; olai's `main.tsx` becomes `followBundle(...)` | drishti pair PR (surface-app option) |
| 3 | reference rows `ws`, `web-app`, `unix-socket`, `stdio`, `mcp-endpoint`; the counter example | olai's `ws`/`web-app` rows become one-line re-exports; example runs the three ways of §7 in kolu CI | none |
| 4 | second consumer | candidate: drishti's app shell as rows (it already consumes every surface package and has a host map to publish as siblings), or kolu's own Debug/Inspector panels as rows on kolu's server | the human's ruling — see §9 |

Each stage is one PR; none is implementation approval. Stage 0 is worth doing on its own even if nothing else lands: it removes the only Cordis-naming package from olai without changing a line of it.

## 9. Questions for the coordinator

1. **Who is the second application?** Without one, the electricity bar says prove-then-extract and stage 1 is a move, not a proof. I do not know of a planned Cordis app besides olai. Is there one? If not, is "olai's own serve.ts shrinking from 349 to ~60 lines" enough for the human, given the atlas's own rule?
2. **Does `system/roster` belong in `@kolu/surface` (as the atlas row says) or in `@kolu/surface-cordis`?** I followed the atlas. It costs a drishti pair PR; the alternative costs every consumer the silent-forget failure.
3. **`Env`/`Clock`/`LocalState`: app or framework?** I left them app doors. `Clock` is arguably framework (a test injectable every app wants). Cheap to move later.
4. **Naming.** `@kolu/effect-cordis` keeps olai's name because the fence and the docs already use it. If kolu wants one prefix, `@kolu/surface-cordis/runtime` as a re-export door is fine; the package boundary should stay.
5. **Where does the shutdown "tenure" live** — `serveBundle` here, or the deferred shared sequence in `@kolu/surface-daemon` that the lifetime audit names? I put it here because the daemon package is about a durable process with an upgrade window, and a bundle serve is not necessarily that.
6. **Per-sibling fault channel** (§6.9) — is a framework ask worth opening now, since a plugin host is the first consumer that needs "one broken plugin is one absent plugin" to hold past `apply`?

## 10. What I did not read

`surface-cordis.html`, `surface-cordis.pages.dev`, `codex.md`, any other proposal. `docs/architecture.md` beyond its first 51 lines and the `plugin-system.md` sections not cited. Cordis upstream source only through the research pass (`context.ts`, `fiber.ts`, `reflect.ts`, `events.ts`, the loader's `tree.ts`/`entry.ts`) and olai's assumption table; the paper via its PDF text, §3–§5.
