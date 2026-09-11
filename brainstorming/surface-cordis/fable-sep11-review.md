# fable-sep11-review — the settings/search delta against the updated consolidation.md

Token `CORDIS-SEP11`. Independent review of olai `ce5f233e1 → 5e087668e` (#565 cordis doc, #566, #569 settings in the vault, #576 search UX, #577 docs relocation) read from the diff, the new sources and their tests, then compared with `consolidation.md` as saved 2026-09-11. No other reviewer's notes seen. PROPOSED code does not compile.

## 1. What the delta actually is

| mechanism | where (olai `5e087668e`) | one line |
| --- | --- | --- |
| plugin declaration | `effect-cordis/src/plugin.ts:180-197`, `:214-228` | `definePlugin` gains `environment?`, `config?` (an Effect schema with no decoding services), `configUpdates?: "reapply" \| "live"`; config is decoded by `Schema.decodeUnknownSync` inside the activation, so a bad value fails the row before `apply` acquires anything |
| loader patch | `effect-cordis/src/loader.ts:246-262` | `patchRow(host, id, {disabled?, config?}, force)` compares canonical JSON of decoded config to the entry's options; unchanged → no-op; changed → interrupt, `entry.update`, wait out inertia; never `tree.write()` |
| service watch | `effect-cordis/src/host.ts:128-139` | `serviceChanges(host, key)` — a root-only stream of "who offers this key now", off the engine's `internal/service` event |
| the worker | `server/src/configuration.ts` (130 lines) | `followConfiguration`: one queue; `switchMap` over `serviceChanges(ConfigurationSource)`; per publication: drop if the source is no longer the offered one, strip `on` for session owners, one `patchBundleRows` for every built row under `Effect.uninterruptible`, one settle, one `changed()`; `awaitRevision` refuses settlement if the reader withdrew; `set` has a session fallback, `configure` has none |
| the provider | `plugins/settings/src/server.ts` | a row: `needs [Vault, BundleModules, Offers]`, re-reads `_olai/Settings.olai` on every vault revision, `offers.offer(ConfigurationSource, …)` — publishes data only |
| bootstrap | `bundle/src/bundle.ts:290-321`, `serve.ts` | `mountBundle(host, [], profile, prepare = true)` mounts with the `test-minimal` profile patch stacked on top (the bootstrap set), then `followConfiguration`'s first publication — or an immediate `source: undefined` on a profile with no reader — writes the real `disabled`/`config` patches for all rows, then `policy.ready` |
| authority | `plugins/settings/src/policy.ts`, `transport.ts:85` | `writeReservations` gains `file?`; `on` in the settings file is reserved against the agent face through the static `/policy` door, in force while the reader is off |
| live consumer | `plugins/kolu/src/server.ts:226-283`, `kolu/src/config.ts` | the one `configUpdates: "live"` row ignores `apply(_settings)` and re-decodes its own namespace from the vault revision itself with `decodePolicy` |
| optional UI | `plugins/search/src/browser.tsx:54-66`, `contracts/box.ts` | `kind` component owns a `createRoot` signal and offers `search.kind`; `selector` component needs it and registers into `search.box.below`, a child location the header contribution declares (`children: [boxBelow]`), so it exists only while the header entry is active |

## 2. Where consolidation.md is right

Config decoded before acquisition; identical decoded policy preserves activation; the loader reconciles without writing; a serialized host worker that checks provider identity, finishes accepted patches regardless of the publisher's lifetime and never rolls options back on withdrawal; write and apply as separate outcomes with the "persisted, not confirmed" refusal; enablement and configuration authorized separately; bootstrap providers keep a recovery path; storage, discovery, recovery rules and UI stay plugins; secrets through environment declarations, never config; browser interaction state owned by its activation; the stop order (`policy.close` registered after `plugins.close`, so it runs before it, `serve.ts`). All of that matches the source and its tests (`configuration.test.ts:46-135`).

## 3. Omissions and one overreach, with the correction

### 3.1 `live` today is "re-parse the file yourself"; the recipe should mint a keyed policy door (OVERREACH → PROPOSED)

Consolidation §2 shows a live plugin with `needs: [WatchPolicy]`, "an app-owned narrow service with revision subscription". No such service exists. The only live row re-reads the raw settings nodes on every vault revision and decodes its own namespace with the shared helpers (`kolu/src/config.ts:12-22`, `server.ts:228, 283`). Two readers decode one namespace — the settings row for the panel, the plugin for itself — which is the two-opinions-of-one-value class the format package's header is a list of, and a plugin that can read the whole file can read another row's namespace. The worker never patches a live row's config, so its `apply` sees the schema default and nothing else (`configuration.ts:70`, `bundle.ts:301-305`).

The grounded fix is the stamp the bridge already has for every keyed door: a per-plugin provision minted from the fiber name, fed from the host's single decoded reading.

```ts
// @kolu/surface-cordis/server — PROPOSED; the provision shape is Provision<Shape> = (plugin) => Shape (service.ts:67)
export const OwnPolicy = serviceTag<{ current(): Config; changes: Stream<Config> }>("policy")
yield* provide(host, OwnPolicy, (plugin) => ({
  current: () => worker.rowConfig(plugin),                 // the host's decoded reading for MY namespace only
  changes: worker.rowChanges(plugin),                      // fed from ConfigurationSource; `undefined` reader → last applied stands
}))
// a live plugin:
definePlugin({ name, config: Config, configUpdates: "live", needs: [OwnPolicy, …],
  apply: Effect.gen(function*() { const policy = yield* OwnPolicy; yield* followPolicy(policy) }) })
```

One decode, one namespace per plugin, no file grammar in a plugin. Consolidation should mark `WatchPolicy` as this proposal, not as an existing pattern.

### 3.2 The bootstrap set is spelled as a test profile

`prepare = true` stacks `profilePatch("test-minimal")` over the selected profile so the first mount is the minimal rows (`bundle.ts:310`), and the first publication then enables the rest. Consolidation §5 says "its bootstrap set and dependencies are app-declared" — right, but the recipe must not inherit the spelling: a profile name used as a bootstrap set is a second meaning on one word.

```yaml
# app.yml — PROPOSED: the bootstrap set is its own row field, not a profile
- id: settings
  name: my-settings/server
  bootstrap: true        # mounted before policy is read; its file `on` is ignored (see 3.4)
```

Also record the settings-free path: with no reader row, `serviceChanges` emits `undefined` at once, the worker applies profile/build defaults and `ready` settles (`configuration.ts:47-49, 78-90`) — same code path, no second boot.

### 3.3 Enablement has a session fallback; configuration deliberately has none

`set` falls back to the in-memory flip when there is no reader or the row is a session owner (`configuration.ts:96-100`); `configure` fails with "Settings can be edited when the configuration reader is running" (`:113-116`, test `:124`). Consolidation says "durable/session-only control policy belongs to the app". State the asymmetry as the rule it is: **a switch may be session-only; a value edit is durable or refused**, because a session-only value would be a knob that lies after restart.

### 3.4 Session-only owners are derived from the offers table, and their file `on` is ignored with one warning

The exempt set is whoever currently offers `Vault` and `ConfigurationSource` (`serve.ts`: `plugins.offers().get(…)`), not a static list, so a replacement provider inherits the exemption; the worker strips `on` for them and warns once per file and row (`configuration.ts:60-68`). Consolidation's "recovery path" sentence should say both: derived from live offers; file enablement for those owners ignored, not merely "session-only".

### 3.5 One batch, one settle, one publication per accepted revision

`patchBundleRows` patches every built row then settles once (`bundle.ts:328-334`), under `Effect.uninterruptible`, and `changed()` runs once after (`configuration.ts:86-92`). Consolidation's "serialized worker" should say this, or an implementer settles per row and the tab redials N times for one edit — the same frame-storm the `moving` flag exists to prevent.

### 3.6 Two validation layers, two outcomes

A bad LEAF in the file is coerced to its default with a named problem and the row keeps running (`settings/src/config.ts:12-20`; panel scenario "A refused file value is alarmed inline while its default remains in force"); a malformed FILE defaults every row and marks the reading `broken`, which refuses durable edits until repaired; an invalid DECLARATION (schema that cannot decode `{}`) fails the row at activation (`bundle.ts:304`, `plugin.ts:263-266`). Consolidation's "keep authored, effective, provenance and problems distinct" covers the data; add the three outcomes so nobody makes a bad leaf fail a row or a bad schema silently default.

### 3.7 The static reservation door is a host obligation

`on` in the settings file is refused to the agent face by a `/policy` export the bundle combines at build, independent of runtime selection (`settings/src/policy.ts`, `@olai/bundle/policy`, `transport.ts:85` `file?`). Consolidation §3 has "Writer identity, ticket minting, credentials" downstream and nothing about reservations. The mechanism is generic: a row may reserve keys, files included, and the reservation stands while the row is off — otherwise disabling the policy provider grants the write. Add it beside the write-tag plumbing candidate.

### 3.8 `serviceChanges` is new bridge API and root-only

The worker rides `serviceChanges(host, ConfigurationSource)` (`host.ts:128-139`, exported from the bridge index, reached by the root through `@olai/bundle/bundle`). It is the bridge's first "optional availability at the service boundary" door and belongs in the framework's bridge API list — withheld from the plugin door like `openHost`, since it takes a `Host`.

### 3.9 The host has a namespace of its own

`declarations.set("olai", ProcessConfig)` (`configuration.ts:27`): the serve edits its own logging policy live from the `olai` node, and the roster's `instance` reports address, hostname and their authors (`runtime.ts` diff, `serve.ts` `instance:`). Consolidation §2 treats configuration as a per-row thing. The recipe needs one sentence: the host may declare a policy schema for itself under a reserved namespace, consumed live; what that schema contains (logging) is the app's, and address provenance is a host report, not a setting.

### 3.10 Two small pins

- The roster gained `switchPersistence`, `desiredOn`, `configurationValues` with `setBy`, `configurationAvailable/Error/File`, `environment` (`runtime.ts` diff). These are the app cell's fields and confirm §4's "bundle metadata … do not belong in the generic manifest"; say so in one clause so a reader does not move them onto a future `system/roster`.
- #577 rewrote `docs/architecture/cordis.md` (212+/311−, no facts dropped per the PR) and moved `plugin-system.md`, `slot-ownership.md`, `overview.md` under `docs/architecture/`. Consolidation links the #565 permalink; link the `5e087668e` paths so its anchors resolve.

### 3.11 Search confirms the filter demo, with one detail worth keeping

The optional filter's contribution goes into a CHILD location the header contribution declares at registration (`children: [boxBelow]`, `browser.tsx:47`), so the filter can exist only while the header entry is active, and the header holds the faces it draws through its own `heldFaces` (`faces.ts`). Consolidation §6's demo sentence matches; add "declared as a child of the view's contribution" so the demo proves the child-location rule rather than a plain slot.

## 4. Disposition

No overreach beyond §3.1's `WatchPolicy` being presented as existing. The eleven items above are additions of one to three sentences each, plus the `OwnPolicy` proposal and the `bootstrap:` row field. With them folded the configuration section is a faithful account of #569 as merged. Review closed; no implementation.
