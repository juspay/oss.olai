# fable-cordis-doc-review — `docs/architecture/cordis.md` against consolidation.md

Token `CORDIS-DOC-REVIEW`. Reviewed `/tmp/olai-cordis-architecture.md` (the master copy of `juspay/olai/docs/architecture/cordis.md`; local `ce5f233e1` predates it, but every mechanism it names is in that tree at the cited lines) against `consolidation.md` as saved after the first review round. Independent read; no other reviewer's notes seen.

The doc is a statement of obligations the merged design already keeps (#554, #557). Consolidation.md keeps most of them by extracting the code that keeps them. What follows is the residue: obligations the doc makes explicit that the recipe's sketches do not yet carry, each with where olai keeps it today and the shape the recipe needs. PROPOSED code does not compile.

## 1. Obligations missing from the recipe

### 1.1 Framework-owned dispatch must be gated — `rosterMoved` is a hand-rolled bus

Doc: "Removing a listener from a registry does not prevent a dispatcher from calling an old snapshot … a handler may already have acted on a released resource" (§Stopping). The bridge answers with `gate` + `broadcast` (`effect-cordis/src/gate.ts:1-60`, `broadcast.ts`). But the composition root's own bell to non-wire faces is a bare `Set<() => void>` walked at the tail of `recompose` (`server/src/runtime.ts:239, 350`, `472-475`) — exactly the shape `plugin-api/src/runtime.ts:60-76` warns against ("a registration stops unwinding with its writer … a handler that dies takes the dispatch down with it"). Consolidation §4 says "MCP consumes the snapshot through existing `reroster`" and inherits this bus unchanged.

Obligation: every place the recipe calls plugin code is a `broadcast` (gated, contained, awaited), never a `Set`.

```ts
// @kolu/surface-cordis/server — PROPOSED
export const compositionMoved = broadcast<Composition>("composition")   // EXISTING primitive, broadcast.ts:86
// host, at the tail of recompose, after the gate is read:
yield* compositionMoved.tell(snapshot)                                   // awaits every listener, contains a dying one
// mcp-endpoint row:
yield* compositionMoved.listen(/* stamped by the fiber */)((next) => Effect.promise(() => served.reroster(siblingsOf(next))))
```

The same check applies to `Surfaces.register`'s `published?.(ctx)` callback (`runtime.ts:303`, synchronous, one call at mount — acceptable, but say so) and to `changed` thunks (`services.ts:1540`, synchronous, no plugin code — fine).

### 1.2 A control verb that can close its own connection runs in the host's scope

Doc: "A handler may request disposal of its own plugin: that path must cut and unwind the invocation before releasing the plugin's resources, without deadlocking." Consolidation §4 makes enable/disable "an explicitly authorized control capability" and stops there. Olai's `plugins.set` is the worked case: a flip that turns the ws row off closes the very socket asking for it, so the handler is `Effect.forkIn(runtimeScope)` and the request only `Fiber.join`s (`runtime.ts:188-208`).

```ts
// app-owned control procedure — the obligation is the fork, not the verb (EXISTING shape, runtime.ts:198-207)
set: ({ input }) => control.flip(input.name, input.enabled).pipe(Effect.forkIn(hostScope), Effect.flatMap(Fiber.join))
```

The recipe should hand the app a `HostControl` service whose `flip` already carries the fork, so an app cannot write the verb on the request's fiber. Authorization stays the app's (`authorityAt` on the tag).

### 1.3 The browser host is a module singleton today; the recipe must be a factory

Doc: "Each browser app receives its own `Edits` registry. Two apps must not route edits through one module-global table" (§Owner before helper); "two hosts remain isolated" is named evidence (§Evidence). `openApp` already allocates `Edits` per call (`browser.ts:806-809`), but everything around it in `@olai/web` is module scope: `mounted`, `failures`, `clients`, `composing`, `app` itself, `run = standing()` (`web/src/host/runtime.ts:89-164`), plus `live`, `composed`, `inFlight`, `rows` in `wire.ts`. That is the one-app-per-tab assumption the doc's own rule rejects for general packages ("Moving a shared mutable table into a utility package does not give it an owner", §Imports).

Obligation: `followBundle` returns an instance and owns everything above; two calls in one page do not share a table. Also its dispose must refuse a mount landing after an in-flight roster frame's `await` (§Acquisition: "after loading, check whether the owner still exists before allocating") — today `rerostNow` mounts after `loadRows` and `redial` with no owner check (`wire.ts:230-278`).

```ts
// @kolu/surface-cordis-app/browser — PROPOSED
const tab = await followBundle({ modules, core, composition: readComposition, mount, retired })
tab.readout        // connecting · live · degraded · reconnecting · retired  (EXISTING: live.readout)
tab.reports        // per-row browser activation reports (EXISTING: browserReports, host/runtime.ts:188)
tab.retry; tab.reload
await tab.dispose()   // joins an in-flight roster frame; a frame that lands after this mounts nothing
```

### 1.4 Host readings are declared services, not the composition root's closures

Doc: "consumers read contributions through declared `Faces` or renderer services. They do not import private browser composition machinery" (§Imports); the audit's §3 says the same of `Mounted.tsx` importing `plugins/runtime.ts`. Consolidation §3 keeps "inspector UI" downstream but gives it no door: olai's inspector reads the roster through the app cell that `bind` writes from closures over `offered.report()`, `names()`, `configs()`, `switched()` (`runtime.ts:127-142`, `serve.ts:197-216`), and the tab's `supplyManagement` (`wire.ts:394-402`) is again a module-level hand-off.

Obligation: the recipe provides the readings as host services on both hosts, so an app's core-surface deps and an inspector row `need` them.

```ts
// server (PROPOSED)                                 // browser (PROPOSED)
HostReports = serviceTag<{                           HostTab = serviceTag<{
  composition(): Composition                           readout: Accessor<Readout>
  rows(): ReadonlyMap<string, RowReport>               reports: Accessor<ReadonlyMap<string, RowReport>>
  configs(): ReadonlyMap<string, Config>               retry: Effect<void>; reload(): void
  changes: Stream<void>                              }>("host.tab")
}>("host.reports")
// app core deps become a read over a declared door, not a closure:
cells: { plugins: { store, connect: cell => (yield* HostReports).changes.pipe(Stream.runForEach(() => cell.set(roster()))) } }
```

`HostControl` (§1.2) is the write half; the two are separate keys so a row that only draws never holds the switch.

### 1.5 `Wired` is a broker and its absence/replacement contract is API

Doc: "a narrow, declared broker with explicit absence and replacement behavior, as with `Served` or `Wired`" (§Optional); "surviving consumers retain their client when their loaded module is unchanged … departing client keys are removed and calls to departed capabilities are refused" (§Browser state). Consolidation §2 says only that `Wired.client()` is `unknown`.

Obligation: state the three behaviours as the door's contract, and ship the consumer-side holder the doc requires ("The consumer keeps its own hold … release must remove its own installation", `ui-primitives/src/held.ts:1-60`).

```ts
// @kolu/surface-cordis/browser — contract to write down (EXISTING behaviour, browser.ts:690-708, wire.ts header)
interface Wired { client(): unknown | null }   // null: this wire does not carry my sibling → draw the absent arm
// identity retained across redial iff my module is unchanged; a departed sibling's calls fail with SurfaceSiblingDropped
// consumer hold — the factory, minted privately per package (EXISTING, held.ts)
const wire = heldService<SurfaceClient<typeof surface.spec>>()
yield* Effect.acquireRelease(Effect.sync(() => wire.hold(client)), stop => Effect.sync(stop))
```

`heldService` is Solid and lives in `@olai/ui-primitives`; the recipe's browser door is the right home for it (it is the only generic browser furniture the doc names as an obligation).

### 1.6 Every recipe registry states cardinality and claims atomically

Doc: "For exclusive claims, check and install all entries in one indivisible operation … A refused multi-key claim installs nothing. Its cleanup must not remove the winner's entries" (§Optional). Olai keeps this at three write points: `Edits.register` (`browser.ts:570-588`: one `Effect.sync`, loser releases nothing), `openViews` (`vault/src/views.ts` header: the two-Effect version was reproduced racing), and the bridge's `registry.claim` (`registry.ts:168-187`). Consolidation §4's `Composition` is built from these but does not say the recipe's own tables obey it.

Obligation, as a table the recipe documents and tests once:

| registry | cardinality | claim |
| --- | --- | --- |
| `Surfaces` (one sibling per plugin) | one per owner | `registry.claim`, refused with both names (`services.ts:1541-1551`) |
| `Offers.own(word)` | one per key | Cordis `provide` + `OfferConflict` prose match (`services.ts:1582-1599`; pin assumption) |
| `Locations` | `one`/`many`, `keyedBy` | rival check inside the entry's own activation (`locations.ts:218-220`) |
| listener contributions | many | symbol-keyed, withdraw only own (`listener.ts:150-153`) |
| `compositionMoved` (§1.1) | many | broadcast |

The board sketch's `addCard` is `many`; a shell offering `board.cards` is `one` via `Offers.own` — both already covered if the table is kept.

### 1.7 Adapter acquisition is bracketed, and the reference `mcp-endpoint` row must keep it

Doc §Acquisition names `serveFace`'s `Effect.acquireRelease` around `serveSurfaceAsMcp` (`plugins/mcp/src/endpoint.ts:278-293`), with the stated cost that the acquire is uninterruptible. Consolidation lists the MCP row as an extracted transport without this. One line in the row's contract:

```ts
// rows/mcp-endpoint — EXISTING shape, must survive extraction
Effect.acquireRelease(Effect.promise(() => serveSurfaceAsMcp({...})), served => Effect.promise(() => served.close()))
```

### 1.8 Reconnection semantics are a consumer obligation the shell demo must show

Doc: "Subscriptions return rather than remaining uninterrupted … roughly one-second `pending` gap … A healthy connection readout alone does not establish that a value has resumed updating" (§Browser state). Consolidation §6's job board replaces the board shell but never reconnects. Add: "redial with the board mounted → cards draw `pending`, then resume; no card restarts" — the negative being a card that caches the last value and paints it as live.

### 1.9 The fence ships as a reusable claim set, including live state behind doors

Doc §Refactors: the fence checks "known forms of live state behind public contracts, including aliases and implementation reached through re-exports", and "`const` and a `ReadonlyMap` annotation do not establish those facts". That is `liveStateIn` (`bundle/src/fence.test.ts:2485-2596`) beside the closure walk, with a corpus derived from the workspace list (`tree.testlib.ts`). Consolidation §3 keeps only the import-closure half. Obligation: both claims, parameterised by an app's workspaces and doors, or every consumer re-writes 2,600 lines or skips it.

```ts
// @kolu/surface-cordis-app/testing — PROPOSED, wrapping olai's existing claims
fenceClaims({ workspaces: "package.json#workspaces", engine: "@kolu/effect-cordis", doors: { server: [...], browser: [...] } })
// claims: only `engine` names cordis · no plugin imports another's non-door · browser doors reach no node: · no live state behind a door
```

### 1.10 The pin and its assumption inventory need a kolu home

Doc: "Keep that inventory current when the bridge or upstream pin changes; do not spread private runtime assumptions into feature plugins." Olai hydrates `cordis@4.0.0-rc.9` from a git rev with a per-file `@ts-nocheck` stamp (`nix/cordis.nix:127-173`) and holds the versions with `scripts/check-hydrated-deps.sh`. Kolu's packages are pnpm-declared and baked into consumers via Nix. Moving `@kolu/effect-cordis` therefore needs a decision consolidation does not record: npm release vs the same npins hydration, where `upstream.test.ts` runs, and which repo carries the four upstream asks. Without it stage 0 ("bridge + tests") is not specifiable.

### 1.11 Ranked reads need the rank supplied once

Doc: "`Faces` reads contributions in bundle order, with the rank supplied once by the host" (§Imports). `followBundle({modules})` must derive `rank` from the generated module order and pass it at `attach` (`BrowserMount.rank`, `mount.ts`; `main.tsx:18` today) — one line, easy to drop, and dropping it gives plugins arrival order while the shell gets file order (`browser.ts:721-726`).

## 2. What the doc confirms consolidation.md already has

- Stop protocol order (§Stopping 1-4) = consolidation §5 "stop one activation", kept by extracting `lifecycle.ts:138-192` and the gate tests.
- Component caveat (§Optional: "A component waiting forever … can change the row's reported readiness") = consolidation §2 sentence.
- Broker over locator (`Served`, not `current(anyKey)`) = consolidation §2/§3 "explicit supplied service".
- Detached work owned (`detached`, `detached.held`) = consolidation §6 worker case.
- Durable actions not undone = consolidation §5 last line.
- Only `root` permanent; contributions' `activate` in a separate scope; withdrawal drains dependents = consolidation §2 renderer/board text, kept by `locations.ts`.
- Assumption inventory is the bridge's = consolidation §3 "bridge and its assumption tests" (pending §1.10).

## 3. Vocabulary to adopt

The doc fixes seven words (row, component, activation, service, scope, contract door, broker). Consolidation uses "activation" and "component" already; it should say "contract door" where it now says "contract" or "static contract", and "broker" only for `Wired`/`Served`-shaped services, since the doc makes the word carry a specific obligation (owns its availability policy, is not a renamed locator).

## 4. Disposition against the updated consolidation.md (doc pinned at olai `139366aff`, #565)

Absorbed in substance: §1.5 (`Wired` survivors, resubscribe-with-pending, fresh-data check), §1.6 (atomic claims, buses allow many), §1.7 (release secured across the handoff), §1.8 (reconnect proof), §1.3's two-hosts proof and owner recheck after `await`, and the doc's vocabulary. Remaining, each one sentence or one line of API away:

1. **§1.1 — the composition-moved bell is still a bare `Set`.** The new §5 text ("register each invocation before it can execute, interrupt that invocation rather than its publisher") states the rule; consolidation should apply it to its own dispatch by naming `compositionMoved` a `broadcast`, since that is the one framework-owned call into plugin code the extraction inherits unchanged (`runtime.ts:239, 350`).
2. **§1.2 — the control verb's fork.** "Preserve self-disposal paths" is generic; say that the recipe's `HostControl.flip` carries `Effect.forkIn(hostScope)` so the request fiber only joins (`runtime.ts:198-207`).
3. **§1.3/§1.4 — `followBundle` returns nothing and `composition:` is still a binding.** The doc's rule against reading private composition machinery (§Imports; audit §3) needs the readings to be declared services (`HostReports`/`HostTab`) and `followBundle` to return an instance with `dispose`; otherwise the inspector row and the app's core deps are closures again. This is my open question 1 below and the largest residue.
4. **§1.9 — name the live-state claim.** "Access/lifecycle fences" should say the two claims explicitly (import closure per door; no live state behind a contract door, `fence.test.ts:2485-2596`) and that they ship parameterised, or every consumer skips the second.
5. **§1.10 — the cordis pin mechanism in kolu** is still undecided and gates stage 0.
6. **§1.11 — rank supplied once** at `attach`: one line, absent from the `followBundle` sketch.

Endorsed as plan of record with those six folded; nothing in the updated synthesis contradicts the architecture doc. No implementation.

### Reconciliation — agreed

The coordinator's ruling is accepted on every point: instance-returning `followBundle` with `dispose`; host-owned fork for `flip`; read-only `HostReports`/`HostTab` separate from control; parameterised fence claims; canonical rank; the composition binding as inert contract/cell selection (an owned closure is legitimate — the doc's rule is about module-level live state crossing a wall, not about closures); the pin policy (stage 0 keeps olai's exact pinned source, patches and assumption tests, through kolu's existing npins/pnpm conventions, reproducibility verified before it ships).

On `compositionMoved` the ruling is right and my §1.1 sketch was too quick: `recompose` is synchronous on purpose — a draining key is deferred as a continuation precisely so the pass blocks on nothing (`runtime.ts:276-289`) — and `broadcast.tell` awaits its listeners, so a listener that flips a row would re-enter the pass. The shape to prove, before the `Set` goes, is tell-after-the-pass, forked onto the host scope the way the status queue already is (`runtime.ts:449-466`), with a regression that a listener triggering disposal neither deadlocks nor recomposes twice. Recorded as an audit target, not a reproduced bug. Standing down.

## 5. Two questions for the reconciliation

1. §1.4 puts `HostReports`/`HostControl`/`HostTab` on the recipe as declared services. That is more API than consolidation §1's `composition: appCompositionBinding`. Is the binding meant to be those services under another name, or a callback the app writes? A callback is the closure the doc argues against.
2. §1.10 is a repository decision, not a design one. Who rules on the cordis pin mechanism in kolu?
