# fable-review — comparison with codex.md, and the coordinator's consolidation choices

Token `SURFACE-CORDIS-FABLE-20260908`. Written after `fable.md` (left unchanged as the independent original) and after reading `codex.md` and `previous-sketch.md`. Citations are to olai `ce5f233e1` and kolu `56152455b` unless stated.

## 1. Where the two proposals agree (no decision needed)

- The bridge is extracted verbatim, tests included, as its own package (`@kolu/effect-cordis`). Both.
- `definePlugin`/`needs`/owner-stamped `Offers` are the plugin contract on both halves; no new callback DSL, no `defineServer`/`defineBrowser` (the archived sketch's shape, dropped in both).
- Renderer, shell, transports, navigation are rows. Only `root` (cardinality one) is a location the renderer row owns; `locations()` stays in the bridge; neither Surface nor the host knows a panel name.
- Kolu's rooted bundle, `reroster` and `redial` are the mechanisms; nothing is rebuilt beside them.
- Olai's teardown order (`lifecycle.ts:138-192`: shut admissions → start interrupting → revoke offers and join dependents → join cuts → release resources) and the four-finalizer serve order move with the code and its tests.
- Counter first, then a second demo; slices land one at a time.

## 2. Where they differ, and what I think

| topic | codex.md | fable.md | my view after reading both |
| --- | --- | --- | --- |
| package split | `surface-cordis` = ownership integration (server + browser doors); `-app` = complete recipe (server + browser) | `surface-cordis` = server host incl. serve; `-app` = browser host | **codex's**, see §3.1 |
| roster on the wire | a host-internal `Composition` snapshot with `revision`; adapters ack applied revision; no reserved member | reserved `system/roster` on the root now | **defer the member**, see §3.2 |
| MCP policy | not addressed | endpoint row with a default no-ticket/no-writes broker | **no default**, see §3.5 |
| listener | host owns the port; rows contribute | same | same; the `listen` option is the open point, §3.6 |
| second demo | job board with a slow worker | not proposed | **take it**, see §3.8 |
| typed browser binding | "typed browser wire binding" in `surface-cordis` | `Wired.client()` is `unknown` by design | codex overstates; see §4.1 |
| failure isolation past `apply` | not addressed | a sibling's running source dying is fatal to the bundle (`server.ts:2435`, `:3903-3908`) | must be stated in the consolidation, §4.2 |
| olai consumption | last slice | per stage | per stage, because no shims (§3.7) leaves no other test bed |

## 3. The coordinator's eight choices — confirm or dispute

### 3.1 `surface-cordis` = integration with server/browser doors; `surface-cordis-app` = complete app recipe with server+browser subpaths; bridge separate — CONFIRM

Concrete reason: it is kolu's own precedent. `@kolu/surface-app` already carries `/serve` (Node HTTP) and `/solid` (browser) as subpaths of one package, and the atlas frames the pair as "the wire" vs "the app shell" (`surface-app.mdx:38-41`). A process split (mine) would put `serveBundle` and `followBundle` in different packages while they are two halves of one recipe that must agree on the row grammar; the layer split keeps that agreement inside one package. One condition, because one package with two graphs is exactly where olai has been burned: a Node-only import behind a door a tab opens "does not fail at a boundary claim — it fails at `bun build`" (`effect-cordis/README.md:153-158`), so olai's closure-walking fence (`bundle/src/fence.test.ts`, per-door export lists) must move with the packages and hold each subpath to its graph.

Contents, under that split: `surface-cordis/server` = `Surfaces`, `Offers`, `composeCapabilities` + recompose, `authorityAt`, `HostLoading`, the owned-key grammar; `surface-cordis/browser` = `Wired`, `Offers`, `BrowserMount`, `openApp`; `surface-cordis-app/server` = `serveBundle`, listener, bundle loader + profiles + pin patch, reports; `surface-cordis-app/browser` = `followBundle`, chunk loading, bootstrap; `surface-cordis-app/rows/*` = ws, web-app, unix-socket, stdio, mcp-endpoint; build helper = the row generator.

### 3.2 Defer a reserved `system/roster` schema until a second consumer proves it — CONFIRM, and the critique is right

The schema I sketched (`{key, state, fault?, missing?, browser?: {chunk}}`) does mix four facts: which siblings the wire serves (composition), which modules the build selected (catalog), whether a browser half activated in this tab (a tab fact the server cannot hold — `web/src/host/runtime.ts:166-208` keeps it per tab for that reason), and the switch (policy). A wire client needs only the first plus the chunk URL, and olai's `plugins` cell already carries all four because it is an app cell and may.

Two reasons for deferral that are stronger than "population one":

- A reserved member cannot be gated: kolu keeps `system/*` always reachable on a gated face, on purpose ("gating them off would break the link", `define.ts:612-620`). So a reserved cell is on the agent face too, and a reserved `set` procedure would be a switch every MCP client can press. Olai hands the agent face an empty map for the core surface precisely so it cannot (`@olai/surface/host.ts:109-153`). **`system.roster.set` must not exist**; the switch stays an app procedure under the app's expose map with the app's authorization — which is what olai's `plugins.set` is today, forked into the serve scope (`runtime.ts:188-208`).
- Deferring the member removes every `@kolu/surface` API change from stages 0-2, and with it the drishti pair-PR gate (`.claude/rules/surface.md:13`). Stages 0-2 become pure additions.

What the framework should still carry, so the cassette is written once: codex's host-internal `Composition` snapshot (`{revision, mounts}`) as the value `composeCapabilities` publishes, and in the browser door a `followComposition(readCell, load)` helper that owns the diff-and-redial loop (olai's `rerost` queue, `wire.ts:138-163`) over ANY cell the app names. The one line left to the app — naming the cell — is the droppable kind `electricity.mdx:60` warns about; at population one that is the accepted cost, and the atlas row should be updated with the four-facts critique so the eventual reserved member carries composition facts only. Revision-acknowledgement per adapter: agree with codex's own caveat — add it when a same-name replacement test shows the two booleans (`moving`, `pendingStatus`, `runtime.ts:357-396`) cannot express it; a single generation number would replace them cheaply if so.

Existing `reroster` remains: yes; it is the framework's own door and the endpoint row calls it from `rosterMoved` (`plugins/mcp/src/endpoint.ts:75-89`).

### 3.3 Prefer composition snapshot + explicit app-owned control with authorization — CONFIRM

Same reasons as 3.2. `authorityAt` (`plugin-api/src/authority.ts:10-24`) is the existing shape for "which tags carry a writer" and should be the mechanism the app's switch is attributed through.

### 3.4 Existing reroster remains — CONFIRM (see 3.2).

### 3.5 No default no-ticket/no-writes broker; the application provides policy — CONFIRM, my §6.5 was wrong

A default is a fallback, and the repo rule is "crash loudly if one is absent rather than silently degrading". The better shape is the runtime's own: the `mcp-endpoint` row `needs` an `McpPolicy` key the app must provide or a row must offer; absent, the row sits `waiting` with the key named on the roster, which is visible, where a default is silent. Olai's `Tools.ticket` answering `NO_TICKET` on a serve with no MCP face is the app's decision about the app's door (`services.ts:1621-1631`), not a framework default.

### 3.6 Correct headless wording; no unused `listen` options; no compatibility re-export shims — CONFIRM, with one concrete shape for `listen`

Headless: fixed in `fable.md` §1 after the first draft — "headless" = no browser rows (renderer, web-app, ws) with MCP/unix-socket/stdio rows allowed; "transport-free" = a second profile under which the listener never binds (`listener.ts:95-101`, `serve.ts:302-305`).

`listen` option: I sketched it as "an unused option on a profile with no transport row", which is a knob. The port is genuinely the host's (one port shared by ws, web-app and mcp rows, `listener.ts:1-15`), so it cannot simply move into a row's `config`. Proposed rule instead: the host refuses at boot in both directions — a profile that selects any non-passive contribution with no `listen` given is a fatal boot error naming the row, and a `listen` given to a profile whose rows contribute nothing non-passive is refused as an unused option. No default, no silent either way. (The fuller alternative, a `listener` row that offers the `Listener` door and takes `config: {host, port}`, is cleaner on paper but reintroduces a boot-phase coupling: barrier 2 needs the door to exist before transports settle, and `start` must run after it, `serve.ts:259-268`. Not worth it for the first slice.)

Shims: olai's imports move to `@kolu/effect-cordis` / `@kolu/surface-cordis*` in the same PR, hydrated like every other `@kolu/*` package (`scripts/check-hydrated-deps.sh`); `olai.yml` names `@kolu/surface-cordis-app/rows/ws` directly. Olai's fence claim changes from "only one package names cordis" to "no olai package names cordis" — still an equality (`fence.test.ts`), which is the property that matters.

### 3.7 Extraction/startup ordering and the listener/policy seam — kept as proposed

No change. One addition from the research pass: `serveBundle` must keep treating the composed runtime's `done` as fatal (`watchFault`, `serve.ts:252`), because kolu states it as a law (`packages/surface/src/server.ts:2416-2424`).

### 3.8 Counter proves extraction; job board with a slow worker proves cancellation and independent shell replacement — CONFIRM, with two test conditions

The job board exercises exactly the guarantees `fable.md` §5 lists and the counter cannot: cooperative interruption and `current()` (`lifecycle.ts:129-137`), a shell row leaving while the MCP row keeps serving, `root` cardinality one refusing two shells (`locations.ts:170-172`), and `reroster` on a roster move. Two conditions so the evidence is about the right thing:

- The slow worker must run its job through `detached.held` (`plugin-system.md:215-224`), so the cancellation path proved is the named seam and not a bare `Effect.runFork` — audit finding #9's class.
- "Disable the worker mid-job" must assert what the interrupted job's finalizer wrote (e.g. the job row marked `interrupted` in the jobs collection), not that the job vanished. The bridge records inverses and cannot prove them ("a recorded inverse is not a proved one", `effect-cordis/README.md:252-257`), and codex's own note applies: disposing registrations does not undo emitted writes.

## 4. Two things codex.md should not carry into the consolidation

### 4.1 "Typed browser wire binding" in `surface-cordis`

There is no per-key type for a sibling client once the roster is data: the compiled-in tuple that recovered one was retired with the generated rows, `Wired.client()` is `unknown` by design, and each browser half narrows once at its edge (`bundle/README.md:57`, `web/src/client/wire.ts:242-250`, `test-counter/src/browser.tsx:26`). The consolidation should say "owner-bound sibling client, narrowed by the plugin", not "typed binding". Contract compatibility is a separate, existing check (`isContractVersionCompatible`, major.minor) and codex's "require exact supported identity, fail explicitly" is the right first rule for an independently built browser chunk.

### 4.2 Failure isolation stops at `apply`, and the consolidation must say so

Neither codex.md nor the coordinator's summary states the hardest limit a plugin host inherits: a sibling whose running SOURCE dies after mount is structural wiring death of the whole rooted bundle, root included (`packages/surface/src/server.ts:2435`, `:3903-3908`), which `watchFault` turns into process exit. The bridge contains a dying `apply` and a throwing handler; it cannot contain that. "A broken plugin is one absent plugin" is therefore true at start and at request time and false for a plugin's long-running source. This is a framework ask (a per-sibling fault channel), not something the host may paper over, and the job-board demo should include the negative case so nobody reads the guarantee as broader than it is.

## 5. Final check of consolidation.md — factual and design errors only

Read at the version saved 2026-09-08 after this review's §1-4. The document is sound; these are the points that would mislead an implementer as written.

1. **§1 contradicts §3/§5 on who owns the address.** "Transport rows own their configuration; an app with no listener has no unused `listen` option" — but the port is the host's, shared by ws, web-app and mcp rows (`listener.ts:1-15`), and §5 has the host "provide listener service → … → open requested listeners". A row cannot own the address of a port it merely contributes to. Pick one sentence: either the host takes `listen` and refuses both an address nobody uses and a transport row with no address (my §3.6), or the listener is itself a row and §5's ordering says how `start` is sequenced after barrier 2.
2. **Both entry points are missing the app's root surface.** `serveBundle({bundle, resolve, profile, inputs})` and `followBundle({modules, mount, retired})` have nowhere to pass the app-owned core: the roster cell and the authorized switch (§4 says they stay app-owned), its `deps`, its face maps, and the label the browser health readout uses (`wire.ts:34, 54`: `core: {surface, name}`). Olai's `bind` takes them (`runtime.ts:169-224`); the recipe must too, or it silently reintroduces a framework-owned roster.
3. **§2 "Independent components may declare different `needs` without blocking their entire module" needs its caveat.** A row's report folds its components, so a component `waiting` on a provider that will never exist makes the whole row read `waiting` and the tab does not load it. Olai's ruling: a component is for a half that is optional to HAVE, never for a provider that is optional to EXIST; that case is a narrow broker (`plugin-system.md:1461-1486`). One sentence avoids the trap the consolidation currently invites.
4. **§5 startup omits the fault watch, which is ordering-critical.** `watchFault` must hold the composed runtime's `done` BEFORE the listener door is published, "or the one settle that matters happens with nobody reading" (`serve.ts:217-220`); and `done` is fatal by kolu's law (`packages/surface/src/server.ts:2416-2424`). Add "watch runtime fault" between "compose surfaces" and "provide listener service".
5. **§5 shutdown order nit.** "close composed surfaces and host": the host closes as part of "drain plugin rows" (`plugins.close` is `closeHost`, `services.ts:1679`), and the composed surface closes LAST, after the rows (`serve.ts:245-248`). The listener door's provision withdraws between the port stop and the rows drain (`serve.ts:243-244`). Reorder the last line.
6. **§4 `faces: FaceExposure` is per face, not one.** Olai publishes one grant per face name per mount (`composition.ts:116`: `Record<face, {universe, tags}>`); the snapshot should carry the map. And `contract: ContractIdentity` has an existing home the text asks for: a sibling's own `identity` via `MountSurfaceOptions` (`server.ts:3933`) carrying `contractVersion` (`identity.ts:78`), compared with `isContractVersionCompatible`.
7. **§4 browser reconciliation must keep the header-policy refresh.** A roster change can change upgrade-header policy without changing the surface map, and the framework's redial skips an unchanged map — so olai forces a socket refresh in that case (`wire.ts:252-277`). "Uses existing link/redial machinery" is true only if that step moves with it.
8. **§6 "replace jobs → old handles cannot reach replacement state" is already kolu's guarantee**, not new work: channels are namespaced per mount generation (`server.ts:4120-4128`) and a stale `ctx` refuses with `SurfaceSiblingDropped` (`:3893-3902`). Write the test as a regression guard, not as a feature.
9. **The hardest limit is still unstated.** Nothing in consolidation.md says that a sibling whose running source dies after mount ends the whole bundle (`server.ts:2435`, `:3903-3908`; my §4.2). §6's job board should include that negative case so "a failure is visible and scoped" (§5) is not read as covering it.
10. **§3 table: "Write attribution" is half generic.** The mechanism — declared `writes` tags wrapped to carry a caller context (`authority.ts:10-24`) — extracts; the writer identity and ticket minting stay downstream. Split the row.
11. **§1 "browser.ts" is a `.tsx`-free entry but §2 mounts JSX**; fine, only note that the browser subpath's graph must be fenced from the server subpath's (`node:`, YAML) per §3.1 above, or the first `bun build` of the tab fails (`effect-cordis/README.md:153-158`).
**Disposition.** consolidation.md is a sound synthesis and I endorse it as the plan of record for the brainstorm: the package split, the deferred reserved member, the explicit policy service, the ordered recipe and the two demos are all right. The coordinator has since accepted the corrections from §1-4 (owner-bound unknown client, host-side shared-address validation, per-slice olai migration, import-closure fences, `detached.held` plus interrupted-finalizer proof, the daemon-boundary note) and confirmed from the Surface source that connector/install and sibling-teardown faults are fatal to the rooted bundle while periodic cell failures are not, adding the limitation and its negative test. The coordinator has since applied items 1-11 in consolidation.md (root surface, deps and faces on both entry sketches; the component caveat; the fault watch before the listener door; the shutdown order; per-face grants with `contractVersion` as the compatibility baseline; the forced header-policy refresh; the replacement test labeled a regression guard; the fatal-source limitation with its negative test; write-tag plumbing split from writer identity). Nothing remaining contradicts the extraction plan in fable.md. **Final disposition: endorsed as plan of record, review closed.** This document authorizes no implementation.

## 6. One open question I still hold

Where the shutdown "tenure" lives — `serveBundle` here, or the deferred shared sequence kolu's lifetime audit names for `@kolu/surface-daemon` (`surface-lifetime-audit.mdx:85-100`). I put it in `-app/server` because a bundle serve is not necessarily a durable daemon with an upgrade window. Cheap to move later; worth a sentence in the consolidation so the daemon owners know a second spelling exists.
