# Settings doors (#545), v2: the vault is the settings file

Grounded in this tree at `5578e3514` (= `origin/master`). v1 (host-composed, CLI/nix kept) is at `/tmp/settings-doors-design.md`; this supersedes it on the human's direction: **stop supporting CLI and nix options; settings live in `_olai/*.olai`; env is for what cannot be in git.**

Where master already differs from the issue's table: `OLAI_TOKEN` is read only by the `olai surface` client (the server mints its bearer per process, `serve.ts:87`); `OLAI_ODU_BIN` and the `OLAI_ACP_*` defaults are baked by the nix wrapper (`default.nix:200-205`), no operator sets them; `--without-plugins=chat` exists, so `OLAI_ACP_AGENT=""` is a duplicate off switch; the login header cannot be emptied (`who/config.ts:97`) though `docs/running.md:128` says it can.

## Three doors

| door | what | authored | restart returns to |
|---|---|---|---|
| **vault** | every policy and behaviour knob, and which plugins run | `_olai/Settings.olai`, one top-level node per row; in git | the file |
| **env** | secrets and machine paths only | unit / `environmentFile` / the nix wrapper | env |
| **browser** | how this tab reads | `⚙`, localStorage | itself; never on `⧉` |

Plus `memory` (`LocalState`, `$XDG_STATE_HOME/olai/<plugin>/<hash>.json`): named on the panel, never opened.

What stays on the command line because it precedes the vault: the directory, `--host`, `--port`, `--profile`. Nix keeps `dataDir`, `host`, `port`, `environmentFile`. Removed: `--commit`, `--push`, `--no-commit`, `--plugins`, `--extra-plugins`, `--without-plugins`, and the nix options `commit`, `push`, `plugins`, `extraPlugins`, `withoutPlugins`. `OLAI_LOG_LEVEL` and nix `logLevel` go too (the `This serve` foot ships here). `olai.yml` keeps `id`, `name`, `section`, `disabled` (the build's opt-in default) and drops `config:`.

Env, exhaustively: `OLAI_SPACES_TOKEN` (secret), provider keys agents read (secret), `OLAI_SPACES_URL`, `OLAI_AGENT_PATH`, `PADI_SOCKET`, the wrapper-baked `OLAI_ACP_*` / `OLAI_ODU_BIN`, `OLAI_ALLOWED_ORIGINS`, `OLAI_HOSTNAME`. Identity header names move to the vault (open question 3).

## The file

```jsonl
# _olai/Settings.olai
{"id":"git","ord":"a0","title":"git","custom":{"commit":"auto","push":"off"}}
{"id":"identity","ord":"a1","title":"identity","custom":{"login-header":"X-Auth-Request-User","picture-header":""}}
{"id":"kolu","ord":"a2","title":"kolu"}
{"id":"kw","ord":"a0","parent":"kolu","title":"watch","custom":{"held-for":"60s","nag":"10m/4"}}
{"id":"journal","ord":"a3","title":"journal","custom":{"on":"no"}}
{"id":"olai","ord":"a4","title":"olai","custom":{"log-level":"debug"}}
```

One file, found the way `_olai/Kolu.olai` is (`kolu/src/config.ts:106`: basename, case-folded, shallowest wins). A top-level node is a row's namespace: its title is the row id, its properties are the knobs, `on` is enablement, and its children are the plugin's own sections (kolu's `watch` moves in; `Kolu.olai` is deleted, no compatibility read). `olai` is the serve's own. Absent node, absent key, or a malformed value = the default, said once on the console at warn (Kolu's rule, `config.ts:143`). No file = every default, which is today's `olai web ~/notes`. One reader in core; a plugin that follows edits live (kolu) reads its own node off the same revision.

Per-plugin files (`Git.olai`, `Journal.olai`) were the alternative: they isolate a bad line to one plugin, but they multiply doors, split kolu's convention from everyone else's, and answer "how is this serve set" across N pages. The cost kept: one bad line defaults every plugin until fixed, and the panel names the broken file the way a broken outline is named today.

## One schema per plugin, and nothing else

```ts
export const Config = Schema.Struct({
  commit: Schema.Literals(COMMIT_MODES).pipe(
    Schema.withDecodingDefaultKey(Effect.succeed(COMMIT_DEFAULT)),
    Schema.annotate({ description: "when a write is recorded" })),
  push: ...
})
export default definePlugin({ name, needs, config: Config, apply: (config) => ... })
```

The schema is the declaration: keys, defaults, prose. The panel's reading is derived from it plus the file, so there is no second list to hold equal. Secrets and machine paths are *not* `Config` fields: they arrive through a keyed provision at activation (like `LocalState`), because Cordis's loader can dump entry options back over `olai.yml` (`loader.ts:186`) and a secret must never be in one.

## How it mounts (Cordis)

1. Rows mount from `olai.yml` as now, on the build's defaults; the vault row opens the directory.
2. A **`settings` row** (`olai.yml`, section Vault, `needs: [Vault]`) parses `_olai/Settings.olai` off every revision and **offers** a `Settings` service: the live value, one entry per row node. It holds no loader verb; a plugin may not (`bundle.ts:59-68`).
3. The **composition root**, which alone holds the loader verbs, subscribes to `Settings` and applies **patches over rows**: `entry.update({ disabled })` for `on`, `entry.update({ config })` for knobs, waiting out inertia and settling as `flipRow` does. Cordis re-applies a row whose config moved; kolu may instead follow its node live off the same revision.
4. The roster publishes, per row, `settings: [{ key, value, setBy: "vault" | "default", says }]` beside the live state. Desired state is the file; actual state is the roster; the panel draws both and conflates neither.

Optional availability: no vault, or the `settings` row off, means every default and a switch that is session-only, said on the row. Reconnection: the vault returning re-publishes and re-patches. Patches are applied and never withdrawn, so toggling the vault does not restart every row. Boot: rows whose file value differs from the build default re-apply once after the vault opens; a pre-vault peek at the file from disk was rejected as a second reader outside the vault's codec and lock. The switch is a write plus a follow: `plugins.set` writes `on`, then awaits the next `Settings` publish and the settle before the strip unfreezes.

## `⧉`

```
Vault                                                       4 on
  vault      format olai                                      [on]
  git        commit auto ·vault   push off                    [on]  ↗
  identity   login-header X-Auth-Request-User ·vault          [on]  ↗
             ▸ 3 at their defaults
Appliances                                                  2 on · 1 off
  kolu       socket /run/user/1000/padi.sock ·env   held-for 60s  nag 10m/4 ·vault ↗   [on]
  xyne-spaces  OLAI_SPACES_URL unset   OLAI_SPACES_TOKEN unset                          [on]
  journal    Off — _olai/Settings.olai says on: no.                                   [off]
This serve                                                 (collapsed)
  host 127.0.0.1 ·flag   port 7714 ·flag   log-level debug ·vault   origins (none) ·env
Settings live in _olai/Settings.olai and travel with this directory.
```

Shows every knob with its author; secrets as set/unset only; `↗` opens the node in the outliner. **The switch writes `on` to the file** through the ordinary write door, so a flip is durable, in git, and the "session-only" foot line goes away; the `vault` row is the one exception, session-only as today, and its row says so. Refuses: secret values, browser prefs, memory contents, any plugin name in code.

## The constraint this argues down, and what holds it up

**"A vault must not say which tools the host runs"** (`pluginPolicy.ts:25-32`, `docs/running.md:396`). Argued down on the human's ruling, with three supports:

- The vault already authors *code* the host runs, after a person approves it (`plugins.approve`). Authoring *settings* is strictly weaker.
- A vault that switches on a tool the machine lacks gets "not here" (`NotHere`), the state a machine without the tool is already in. Nothing is broken by a foreign `on: yes`.
- **An agent still cannot flip a plugin.** `on` on a `_olai/Settings.olai` node is reserved against the agent face the way `approved:` is (`vault-plugins/src/policy.ts`, `ops/src/door.ts:71`, `WRITE_RESERVATIONS`). A person edits it, in the outliner or with the switch.

Kept: instance vs browser; secrets out of git; nothing hidden in `$XDG_STATE_HOME`; `olai.yml` the only place a plugin is named; the inspector walks and hardcodes nothing. Retired: "no settings file that survives restart" (the objection was to an *invisible* file in a state dir; this one is an outline in git the panel links to), and `OLAI_ACP_AGENT=""` as a chat-off switch (`on: no` on the chat node).

## Rules a reviewer checks

| rule | pinned by |
|---|---|
| every `Config` field has a default and a description annotation | `bundle/src/settings.test.ts` (imports every row's module, as `declaredKinds` does) |
| no `olai.yml` row carries `config:`; `disabled` is the only build default | `bundle/src/rows.test.ts` |
| secrets and machine paths are never entry options; a secret reading has no `value` | `effect-cordis/src/plugin.test.ts`, `surface/src/plugins.test.ts` |
| `on` in `_olai/Settings.olai` is refused to the agent face, both directions | `ops/src/door.test.ts` |
| a malformed value defaults and is said once | `plugins/settings/src/config.test.ts` |
| the `Settings` offer is withdrawn with its row; patches applied stand; a returning vault re-patches | `server/src/serve.test.ts` |
| the switch writes `on` and the row follows on the next revision | e2e `settings_file.feature` |
| `web --help` lists exactly directory, `--host`, `--port`, `--profile` | `server/src/main.test.ts` |
| every env door a row declares is named in `docs/running.md` | `tests/plugin_docs.test.ts` |
| general packages name no plugin | `fence.test.ts` (unchanged) |

## Plan

1. **Schema is the declaration**: defaults and descriptions on git, vault, identity, chat, kolu, xyne-spaces schemas; `Plugin` exposes `config`; drop `config:` from `olai.yml`.
2. **The reader**: `olai-plugin-settings` (a new row; parse, `Settings` offer, malformed-value warnings) and `packages/server/src/serve.ts` subscribing and patching; `ops` write reservation on `on`; roster `settings` readings; `pluginState` gains no new word (`off` now names the file). Kolu: `config.ts`'s `koluFileIn` goes, `watchConfigIn` reads the `watch` child of the `kolu` node off the same revision, the wrench opens that node, `the_feed_opens_its_config.feature` and `format.md:322` follow.
3. **Panel**: chips, author marks, disclosure, node link, switch writes the file; foot line reworded.
4. **Remove the flags and nix options**; `gitPolicy.ts` and `pluginPolicy.ts` shrink to the sentences the panel needs; nix module keeps four options; `nix/home/check.nix` updated.
5. **Tests migrate**: e2e `@pin:commit=auto` and every `--plugins=` fixture become a `_olai/Settings.olai` in the scratch vault (`packages/tests/support/hooks.ts`, `world.ts`); `serve.testlib.ts`.
6. **Docs**: `running.md` (the flags, the git policy, "which integrations", the env table), `plugins/git.md`, `plugins/identity.md`, `plugins/kolu.md`, `dynamic-plugins.md`, `format.md` (the `_olai/` conventions), `index.md`. Doc drifts fixed on the way: `running.md:128`, Spaces vars, `OLAI_TOKEN`, `packages/state/README.md`.

In scope for this PR, no deferrals: the `This serve` foot, and a definition (vault-defined plugin) reading its knobs off its own node. Nothing in this design is transitional or documented for later; the review accepts neither.

Cost to name: step 5 is the bulk of the diff; every e2e scenario that passes `--plugins` today is rewritten, and `just run` / `serve` recipes lose their flags.

## Decided (the human, this thread; reversible by saying so)

1. One `_olai/Settings.olai`, a top-level node per row. `Kolu.olai` is deleted outright; no backwards compatibility.
2. The switch writes `on` to the file. Session-only flips are retired.
3. Identity header names live in the vault. A vault served from two machines behind different proxies is not a case this design serves.
4. A config edit re-applies the row (Cordis's own reconcile). A plugin that wants to follow an edit without a restart reads its own node off the revision, as kolu does; that is the plugin's choice, not a rule.
5. Only `on` is reserved against the agent face. Behaviour knobs stay writable by an agent, as `Kolu.olai`'s were.
6. `--profile` stays on the CLI: it picks the process face before any vault is read.
7. The `vault` row's switch stays session-only, the one row whose flip is not written; the panel says so on that row.
8. The `This serve` foot ships in this PR; `OLAI_LOG_LEVEL` and nix `logLevel` go with it.
9. `olai.yml` keeps `disabled: true` as the build's opt-in default; `on: yes` in the vault turns such a row on.

## Editing settings on the panel (added 2026-09-10, ships in #569)

Grounded in #569 at `92bea9293`. The human's two rulings, from the panel as it stands: the row reads like raw fields (`commit manual ·default`, `2 at their defaults`) and its switch is ambiguous beside them; and the panel shows policy but offers no way to change it short of finding `↗` and editing properties by hand. So: **every knob a row declares gets a control derived from its `Config` schema, and each edit is written to the row's node in `_olai/Settings.olai` through the ordinary write door.** The file stays the source of truth; the panel is a face on it, not a second store.

### The row

```
Vault                                                        4 on
  git      Commit: Manual · Push: Off — using defaults     [on] ▸
  kolu     Watch held for: 60s · Watch nag: 10m/4 · +1     [on] ▾
           Watch held for   [ 90s        ]  set in Settings.olai  ↺ Use default
                            how long a terminal holds attention before a report
           Watch nag        [ 10m/4      ]  default
           Watch heartbeat  [ 5m         ]  default
           socket /run/user/1000/padi.sock ·env
           Open settings node ↗
  journal  Off — _olai/Settings.olai says on: no.           [off]
This serve                                                   ▾
           Log level   ( Debug | Info | Warn | Error )       set in Settings.olai
           Log format  ( Auto | Logfmt | Pretty )            default
           hostname … ·process   host 127.0.0.1 ·flag   port 7714 ·flag   origins (none) ·env   bearer set ·process
```

- **The line** is name, summary, switch, disclosure. The name stays verbatim (`git`: it is the namespace a person types). The **switch is the only control on the line** and is labelled `Enable git` (`aria-label`); that, not a caption, is what removes "does this enable committing?". Nothing else on the line is pressable.
- **The summary** (`pluginSummary` in `rows.ts`, pure) lists the effective value of every leaf in schema order as `Label: Value`, at most four then `+N`; a section's leaves are prefixed with the section label (`Watch held for`). Labels: key with hyphens as spaces, first letter capitalised. Values: choice and boolean values capitalised (`Manual`, `Off`, `Yes`); text and number verbatim. Suffix `— using defaults` when every leaf is at its default; otherwise no suffix, and the hover `title` lists which are set in the file. `·default` is never drawn on the line.
- **Expanded** (`▸`, state in `InspectorState.disclosed`, survives roster updates as today): one line per leaf, label, control, provenance word, then the schema description in muted text under it. Env readings follow, read-only as today (`·env`, `·wrapper`, secrets set/unset). Last, `Open settings node ↗` (same Link, `TESTID.pluginConfigLink`), present only when the node exists; the first edit creates it.
- **Provenance is secondary**: `set in Settings.olai` with `↺ Use default` beside it, or `default`. The chip pills go; `data-set-by` moves to the control's line so the existing pins keep their hook.
- **This serve** draws the `olai` node's two knobs with the same controls; the process facts under them stay read-only.

### Controls from the schema

The panel never sees a schema. `decodePolicy` (`plugin-api/src/configuration.ts`) already walks the AST per leaf; it gains a `control` per `PolicyValue`, derived once on the server and carried on the wire:

```ts
type Control =
  | { kind: "choice"; options: ReadonlyArray<string> }              // Literals: Union of Literal
  | { kind: "switch" }                                              // Boolean
  | { kind: "number"; integer: boolean; min?: number; max?: number } // Number, Int, Union[Int, NumberFromString]
  | { kind: "text"; expected?: string }                             // String, with a filter's `expected`
interface PolicyValue { key; value; setBy: "vault" | "default"; says; control: Control; problem?: { raw: string; why: string } }
```

Any other shape is `text`. A choice with four options or fewer is a segmented control, more is a `<select>`; a switch writes `yes`/`no`; a number input is `inputmode=numeric` with `min`/`max` when known; text and number commit on Enter or blur, Escape reverts. Descriptions come from the same annotation the console uses.

`problem` is the arm that was missing: a hand-written value the schema refused (`held-for: 60`) is today a console warning and a silent default. It is now on the value: the control shows the default in force, and an alarm line under it says `File says "60": spell a number and a unit; using 60s`. The summary counts it (`· 1 invalid`).

### The write

One new browser procedure beside `plugins.set`, same exposure (`host.ts` "tool" map), same seam (`wire.ts` management, `BrowserManagement.configure`):

```ts
plugins.configure({ name: "kolu", key: "watch.held-for", value: "90s" | null }) → {}
```

`followConfiguration` gains `configure(id, key, value)` next to `set`, under the same semaphore: refuse a broken file (`Repair … before changing settings`); refuse when the reader is absent (`Settings can be edited when the configuration reader is running`; there is no session fallback for a knob, because a config patch with nothing in the file would be a second door); resolve the leaf by dotted key in the row's declaration and **decode the value with the leaf's own schema before writing**, so a refused value never reaches the file and the refusal carries the schema's message; then write: `prop` on the row's node for a top-level leaf; for a section leaf, `prop` on the child node titled with the section, created with `add` (`parent` = the row node) when absent; the row node itself created as `set` does today. `null` writes `prop` with `null`, which removes the key (`writing.ts:480`), so "Use default" is a delete and not a written default. Then `awaitRevision` and the follower re-applies exactly as for a hand edit: Cordis reconcile, or live for a row that declares `configUpdates: "live"`. The press stays frozen until then.

Names resolve the way `set` does: bundle rows through the root's declaration map, definitions through the vault-plugins catalog (`catalog.configure`, on the definition's own node, `plugin` and `approved` refused as keys), and `olai` through the process `Config`. Two tabs editing one leaf are last-one-wins, as the drawer's property chips are.

Nothing changes on the agent face: `on` stays the only reserved key (decision 5); an agent edits knobs through the ordinary ops door as before. The vault and settings rows' knobs are editable like any other; their switches stay session-only.

### Rules a reviewer checks

| rule | pinned by |
|---|---|
| every leaf kind in every shipped `Config` maps to a control; choice options equal the literals; a filter's `expected` reaches `text` | `plugin-api/src/configuration.test.ts` over the bundle's real schemas (as `settings.test.ts` walks them) |
| a refused hand-written value carries `problem` with the file text and the schema message; the effective value is the default | `plugins/settings/src/config.test.ts`, `surface/src/plugins.test.ts` (decodes on the wire) |
| the summary: schema order, `Label: Value`, capitalised choice values, four-then-`+N`, `— using defaults` only when all default, `· N invalid` | `plugin-inspector/src/rows.test.ts` |
| the switch is the only control on the line and is named `Enable <row>`; `·default` never appears on the line | `rows.test.ts` and e2e `settings_panel.feature` |
| `configure` decodes before writing: a bad value is refused with the schema's message and the file is untouched | `server/src/configuration.test.ts` |
| a top-level leaf writes `prop` on the row node; a section leaf writes on the child, creating it under the row node; a missing row node is created first | `server/src/configuration.test.ts` |
| `null` removes the property and the value returns to default with `setBy: "default"` | `server/src/configuration.test.ts`, e2e |
| the follower re-applies after a panel edit as after a hand edit; a live row keeps its activation | `server/src/serve.test.ts` (git re-applies; kolu `held-for` keeps registration) |
| broken file and absent reader refuse with their sentences; controls draw frozen with the reason | `configuration.test.ts`, e2e `@rows-off:settings` |
| a definition's knob edit lands on its own node; `plugin` and `approved` are refused | `vault-plugins/src/runtime.test.ts`, e2e `definition_settings.feature` |
| `olai` log knobs edit from `This serve` and apply live without a restart | e2e `serve_settings.feature` |
| an agent still cannot write `on`; it can write a knob | `ops/src/door.test.ts` (unchanged), e2e `settings_file.feature` outline |
| env readings have no control; a secret reading still has no value | `surface/src/plugins.test.ts`, `rows.test.ts` |
| the panel spells no plugin name and no key | `fence.test.ts` (unchanged) |

### Plan (commit order within #569)

1. **Wire and decode**: `Control` and `problem` on `PolicyValue` (`plugin-api/src/configuration.ts`, `decodePolicy` derives both; a shared `coerceLeaf` for the yes/no and number spelling), `surface/src/plugins.ts` schema, `BrowserManagement.configure`. Tests: `configuration.test.ts` over the real schemas, `surface/src/plugins.test.ts`.
2. **The verb**: `configure` in `server/src/configuration.ts`; `plugins.configure` in `runtime.ts` and `host.ts`; `catalog.configure` in `vault-plugins` (`runtime.ts` `policyFor`'s node, reserved keys); `olai` via `process-policy.ts`; `wire.ts` management. Tests: `server/src/configuration.test.ts`, `serve.test.ts`, `vault-plugins/src/runtime.test.ts`, `mcp/route.test.ts` (procedure not on the agent face).
3. **Panel**: `rows.ts` (`pluginSummary`, `labelOf`, `valueLabel`, `controlOf`), `Panel.tsx` (line, `Enable <row>`, details with `Control.tsx`, provenance, `Use default`, `Open settings node`, `This serve` controls), `testids.ts` (`pluginSummary`, `pluginControl`, `pluginUseDefault`, `pluginProblem`). Tests: `rows.test.ts`, `browser-selection.test.ts` if the panel's selection walk moves.
4. **E2E**: new `settings_panel.feature` (enable git, read the summary, pick Auto, ledger shows the write, restart keeps it; kolu `90s` creates the `watch` child, `90` is refused with the message and the file is unchanged; Use default removes the key; a hand-written bad value shows under the control; reader off and broken file freeze); `serve_settings.feature` gains the log-level control; `definition_settings.feature` gains a knob edit and a refused `plugin` key; `settings_file.feature` steps reworded from chips to controls.
5. **Docs**: `docs/running.md` (the panel section at `:240`, the write path), `packages/plugins/plugin-inspector/docs.md`, `docs/git.md:120`, `packages/plugins/kolu/docs.md` (the wrench now opens a panel with controls; `↗` is the advanced door), `docs/dynamic-plugins.md` (definition knobs), `docs/format.md:322` unchanged. `website/index.html:635` mocks the panel with switches only and says the panel writes the same file, which stays true; it is not touched.

Not deferred, not transitional: the chips are removed rather than kept beside the controls; `problem` ships with the controls, not after; `This serve` gets its controls in the same commit as the rows.

## Amendment (human, 2026-09-10): the panel is the prototype

The section above shipped at `f37e35f56` with every control behind a `▸` and a summary sentence per row. The human rejected that picture: "too much text, too vertical, actual pref controls". The approved picture is `./settings-doors-panel.html` (also https://claude.ai/code/artifact/c7728f63-c528-4f00-a440-4b5b5dece368). It replaces "The row" and the layout rules of the section above; the verb, wire, `Control`/`problem` derivation, agent reservation and Cordis boundaries stand.

**The panel.** Near 1:1, two columns of sections, sections collapsible with `N on · M off` on the heading, header `⧉ plugins` with `_olai/Settings.olai ↗` on the right, foot with the legend (● set in Settings.olai, dashed ring session-only) and `memory · $XDG_STATE_HOME/olai`. No sentence anywhere except a failed or waiting row's own reason.

**The row** is a three-column grid: name (with `↗` to its node, visible on hover), knobs, switch.
- Knobs are drawn inline, always, one per leaf: short lowercase key (`commit`, `push`, `held`, `nag`, `beat`, `login`), then the control: segmented for a choice, switch for a boolean, short input for text or number (`6ch`, `9ch` for numbers, `22ch` for headers), unit after a number where the key implies one. No summary line, no disclosure, no `Label: Value`.
- Provenance is a ● after the control, drawn only when the value is set in the file, `title` "set in Settings.olai"; `↺` beside it resets (writes `null`). The schema description is the knob's `title`.
- A refused file value draws the input in alarm with one line under it: `<why> · using <default>`; the row stays otherwise ordinary.
- An off row dims its knobs (still editable); it says nothing. `Off — the file says on: no`, `Off by default …`, `Switched off here …` are retired; `Switch is session-only` becomes a dashed ring on the switch with `title` "session-only".
- Env readings are `key env · unset|value` in the knobs, muted, no control; secrets set/unset.
- The switch is the row's only enablement control, `aria-label` `Enable <row>`, right column, no visible caption.
- `This serve` is a section like the others: `log` row with the two segmented controls, then the facts as one muted line (`hostname · env`, `host · flag`, `port · flag`, `origins`, `bearer`).
- Needs-you rows keep their reason line in alarm/doing colour; nothing else under a row.

### Rules a reviewer checks (replacing rules 3, 4 of the section above)

| rule | pinned by |
|---|---|
| every leaf's control is in the DOM with the row, no disclosure between them; no `plugin-summary`, no `plugin-defaults` | `rows.test.ts`, e2e `settings_panel.feature` |
| the row's prose arms (`file says on: no`, `Off by default`, `Switched off here`, `session-only`) draw no text; session-only is the ring's `title` | `rows.test.ts` (`pluginHint` returns `null` for `off`, `optIn`, `switched`), e2e `the_plugin_switch.feature`, `the_vault_is_a_row.feature` |
| ● appears only for `setBy: "vault"`, `↺` beside it writes `null` | `rows.test.ts`, e2e |
| the panel's box is square within 10% at desktop width and the body never scrolls sideways | e2e `settings_panel.feature` (bounding box) |
| section headings count `on · off`; the header link opens the file; the row `↗` opens the node | e2e (`settings_file.feature` steps reworded) |
| a refused value draws inline under its input with the default named | e2e `settings_panel.feature` (kolu `nag` `10`) |

### Plan (one commit in #569)

1. `Panel.tsx`, `Control.tsx`, `rows.ts`: the grid above; delete `pluginSummary`, `labelOf`/`valueLabel` uses on the line, the disclosure and `TESTID.pluginSummary`/`pluginDefaults`; add `TESTID.pluginKnob`, `pluginSource`, `pluginReset`; `pluginHint` returns `null` for `off`, `optIn`, `switched`; `rowCopy` drops the session-only sentence; `popover.ts` summary-in-tab-order change reverts if nothing else needs it.
2. `PANEL_BOX`/anchor: the plugins panel gets a wide square box (`min(880px, 100vw - 2rem)`, `aspect-ratio: 1`), two columns, sections collapsible with state in `InspectorState.opened` as now; phone falls back to one column, no aspect.
3. Tests: `rows.test.ts`, `settings_panel.feature` rewritten to the controls-in-view picture (Tab from a switch reaches the next knob, not a disclosure), the reworded steps across `settings_file`, `plugin_schema_defaults`, `pinned_git_policy`, `the_vault_is_a_row`, `plugin_flags`, `the_feed_opens_its_config`, `serve_settings`, `definition_settings`.
4. Docs: `plugin-inspector/docs.md`, `docs/running.md` panel section, `kolu/docs.md` wrench sentence; `website/index.html:635` mock now draws switches only against a panel that draws controls, so it is redrawn to match the prototype.

Pending rows also draw no status sentence: their Approve block is the call to action, and `rows.test.ts` pins `pluginHint` returning `null` for `pending`.
