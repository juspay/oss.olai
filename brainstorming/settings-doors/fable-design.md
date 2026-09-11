# Settings doors: one map, one reading, one home for defaults

Design for juspay/olai#545 ("Plugin settings live in too many doors, and the inspector only sees one of them"). Design only; nothing in the repository is modified. Written against `origin/master` at `5578e3514`, after #543.

Everything below was read from the tree, not from memory. File:line references are to that commit.

## 0. The diagnosis in one paragraph

A plugin's knobs are already split along real lines (host vs vault, policy vs resource, instance vs browser, durable vs session), and the tree argues each split well in place (`packages/server/src/pluginPolicy.ts:18-38`, `docs/running.md:341`, `:396`). What is missing is not a new door but a **declaration**: no plugin says, as data, *which knobs it has, which door each is authored at, and what its default is*. Because nothing declares, the roster can only publish what the loader happens to hold (`rowConfigs`, `packages/effect-cordis/src/loader.ts:276-294`, "before the plugin's schema folds defaults in"), the panel can only draw that (`Panel.tsx:419`), and the defaults for git had to be copied into `olai.yml` so the panel had something to draw (#543). Identity, the engines, PATH, Spaces, Kolu's pacing and LocalState never appear, and `--help`, nix, docs and code each spell defaults on their own.

The design therefore adds one thing and moves one thing:

- **Adds** a per-plugin `settings` declaration on `definePlugin`, and one `settings` reading per row on the roster: `key, kind, door, value, setBy`. The host fills the readings for the doors it owns (bundle, flag, env); a plugin fills the readings for the one door only it can read (a vault file). Core knows no plugin's words; they travel as data, the way `wake` and `config` already do.
- **Moves** defaults to one home: the plugin's own `Config` schema (which for git already reads `@olai/format`'s constants). `olai.yml` `config:` goes back to being what the header at `packages/bundle/olai.yml:26-32` says it was before #543: a build's overlay, absent in the ordinary case. `--help`, nix option descriptions and docs are then pinned to that home by tests rather than being homes themselves.

Everything else in the issue's inventory is classified by the taxonomy in §1 and drawn, linked or refused by the rules in §3.

## 1. The taxonomy: five doors and one attribute

The smallest set that keeps every distinction the constraints rely on is **five doors** plus **one attribute**. A door is *who authors it and how long it lasts*; the attribute is *what the panel may show*.

| Door (`kind`) | Who authors it | Where it is spelled | Lifetime | Restart comes back to |
|---|---|---|---|---|
| `policy` | the operator, about this **serve** | plugin default → `olai.yml` `config:` → `--flag` / `services.olai.*` → (enablement only) the panel switch | the process | flag / nix / YAML / default |
| `resource` | the operator, about this **machine** | `OLAI_*` env, in the unit or `environmentFile` | the process, read at boot | the same env |
| `vault` | the **directory**, about how an integration behaves here | a file or property in the served tree (`_olai/Kolu.olai`, `_olai/Properties.olai`, `xyne-channel`, `plugin:`/`approved:`, a definition node's properties) | travels with the directory, re-read per revision | the file |
| `browser` | the **reader**, about this tab | `localStorage` via the preferences panel | this browser | this browser |
| `memory` | the **plugin itself**, about this machine × this directory | `LocalState`, `$XDG_STATE_HOME/olai/<plugin>/<hash>.json` (`packages/state/src/index.ts:189`) | machine × directory | the same file |

Attribute: **`secret`**. A secret is always a `resource` (env-only, never in the vault, never in argv, never on the wire). It is an attribute and not a sixth door because its author, spelling and lifetime are exactly a resource's; only what may be *shown* differs. Collapsing it into a door would either duplicate `resource` or lose the "set / unset, never the value" rule, which is the one thing a secret needs.

Why five and not fewer:

- `policy` vs `resource` is the flag-vs-env line the tree already keeps (`pluginPolicy.ts:18-22`). A policy belongs on `--help` so it can be read without knowing it exists; a resource names something the machine has, is set where the machine is wired, is inherited by subprocesses, and may be a secret that must not be in `ps`. Merging them would put either secrets on `--help` or policies off it.
- `vault` vs `policy` is the vault-vs-host line (`docs/running.md:396`). A directory may say how an integration behaves and may not decide what the host runs.
- `browser` vs everything else is why plugins left the preferences panel (`Panel.tsx:10-21`).
- `memory` is not a knob at all; it is a door the panel should *name* (where this row remembers) and never open. Keeping it in the taxonomy is what lets the panel answer "and where does it remember" without pretending a JSON record is a setting.

The issue's inventory, re-labelled:

| What | Door | Secret | Who fills the reading |
|---|---|---|---|
| `disabled:` / `--plugins` family / panel switch | `policy` (enablement; already drawn) | | host (already: `pin`, `state`) |
| `commit`, `push` (git) | `policy` | | host |
| `format` (vault) | `policy` | | host |
| `OLAI_IDENTITY_*` (5) | `resource` | | host, from env |
| `OLAI_SPACES_URL` | `resource` | | host, from env |
| `OLAI_SPACES_TOKEN` | `resource` | yes | host: set/unset only |
| `OLAI_ACP_AGENT`, `OLAI_ACP_CODEX`, `OLAI_ACP_PI` | `resource` on the engine row | | host, from env |
| `OLAI_AGENT_PATH` | `resource` on the chat row | | host, from env |
| `OLAI_ODU_BIN` | `resource` on the odu row | | host, from env (the wrapper sets it, `default.nix:204`) |
| `PADI_SOCKET` | `resource` on the kolu row | | host, from env |
| `OLAI_CHAT_IDLE_MS` | `resource`? No: `policy`, see §5 | | host |
| `OLAI_ALLOWED_ORIGINS`, `OLAI_HOSTNAME`, `OLAI_LOG_LEVEL`, `OLAI_LOG`, `--host`, `--port`, bearer token | the **serve's own**, not a row's (§3.4) | token yes | host |
| `_olai/Kolu.olai` `held-for`/`nag`/`heartbeat` | `vault` on the kolu row | | kolu, per revision |
| `_olai/Properties.olai` kinds | `vault`, core's vocabulary, not a row knob (§3.3) | | not a reading |
| `xyne-channel` on node agents | `vault` on the xyne-spaces row (a count, with no single file) | | xyne-spaces, per revision |
| `plugin:` + `approved:` | `vault`; already drawn as **Defined here** / **Needs you** | | vault-plugins (already) |
| theme, font, size, notes, done, alerts | `browser` | | never travels |
| chat session / model / doorbells | `memory` | | host: path only |
| Kolu socket | `resource` (`PADI_SOCKET`) plus the probe's answer | | host + kolu |

## 2. The declaration and the reading

### 2.1 What a plugin declares

On `definePlugin` (`packages/effect-cordis/src/plugin.ts:201`), beside `config`:

```ts
// @olai/plugin-api/contract.ts — data only, no runtime, both halves may import it
export type KnobKind = "policy" | "resource" | "vault"

export interface Knob {
  /** The plugin's own word for it, and the key in its Config: "commit", "login-header", "held-for". */
  readonly key: string
  readonly kind: KnobKind
  /**
   * WHERE IT IS AUTHORED, as the person who sets it would type it.
   *   policy   → { flag?: "--commit" }            (absent flag = bundle/default only)
   *   resource → { env: "OLAI_IDENTITY_LOGIN_HEADER" }
   *   vault    → { file: "_olai/Kolu.olai" } | { property: "xyne-channel" }
   */
  readonly door: KnobDoor
  readonly secret?: true
  /** One sentence in the plugin's words. Core displays it and composes nothing. */
  readonly says: string
}
```

The `Config` schema stays what the activation decodes (`plugin.ts:245-251`), and it is where the **default** lives, as an Effect Schema decoding default on the field. `browser` and `memory` are not `KnobKind`s: a browser preference is declared where it already is (`preferences.sections`, `docs/plugins/preferences.md`) and never travels to this panel; memory is one host-derived line per row (§3.4), not something a plugin declares.

Fence (test): every `policy` and `resource` knob key is a field of the plugin's `Config` schema, and every `Config` field is a declared knob. A `vault` knob is not a `Config` field (nothing outside the plugin can feed it). A `secret` knob is `resource`. A `policy` knob has no `env` door; a `resource` knob has no `flag` door. These four lines are what make "env names a resource, a flag names a policy" a rule a machine checks instead of a paragraph.

### 2.2 How a field gets its value

The host composes each row's entry options **once, at mount**, from the doors it owns, in this order, last wins:

1. the schema default (folded in by the activation's decode; the host never spells it),
2. `olai.yml` `config:` (the build's overlay),
3. the flag/nix patch (`gitConfigPatch`, `packages/server/src/gitPolicy.ts:186-197`),
4. env, for every `resource` field: `vars[door.env]`, as the raw string or absent.

That is still "a patch applied over rows on the way in" (`olai.yml:41-45`), so `--plugins` and `--commit` keep their exact mechanism; what changes is that the composition root records **per key** which door supplied the value, instead of `gitConfigPatch` writing the whole policy and losing that (`gitPolicy.ts:180-184`). `mountBundle` takes `configs: Array<{ id, config, setBy: Record<key, "bundle"|"flag"|"env"> }>` and keeps the map beside `switched` (`packages/server/src/serve.ts:164`).

**Env leaves the plugins.** Today every plugin reads `Env.vars` itself (`packages/plugins/identity/src/server.ts:54-57`, `xyne-spaces/src/server.ts:65-66`, `chat/src/agents/roster.ts:300`, `claude/src/server.ts:65`, `kolu/src/client/socket.ts:45`). Under the design a `resource` field arrives decoded on `config`, like `commit` does. `Env` stays for the one thing it is still for, `dial` (`services.ts:144-148`), and for a plugin that needs a variable it cannot declare (there should be none; the fence says so). Empty-vs-unset semantics move onto the schema, where they are readable: identity's "unset is the Tailscale name, empty is off" becomes `Schema.optionalKey(Schema.String)` plus a transform, and the one asymmetry the survey found (the login header cannot be emptied, `who/config.ts:90-97`, contradicting `docs/running.md:128`) becomes a visible refinement instead of a doc that is wrong.

Testability is kept: `serve()` already takes `vars` (`serve.ts:43`, `:150`); the composition reads that, not `process.env`.

### 2.3 The reading, on the roster

`BuiltPlugin` (`packages/surface/src/plugins.ts:82`) gains `settings` and loses `config` (the former subsumes the latter; both are `optionalKey`, so an old tab decodes):

```ts
settings: Schema.optionalKey(Schema.Array(Schema.Struct({
  key: Schema.String,
  kind: Schema.Literals(["policy", "resource", "vault", "memory"]),
  /** The door as a person would type it: "--commit", "OLAI_SPACES_TOKEN", "_olai/Kolu.olai". */
  door: Schema.String,
  /** Absent for a secret, and for a resource nothing set and nothing defaults. */
  value: Schema.optionalKey(Schema.String),
  setBy: Schema.Literals(["default", "bundle", "flag", "env", "vault", "unset"]),
  secret: Schema.optionalKey(Schema.Boolean),
  /** A served path the panel can open (vault kind only). */
  link: Schema.optionalKey(Schema.String),
  says: Schema.String,
})))
```

`rosterOf` (`packages/server/src/runtime.ts:40-77`) joins three sources per row:

- **policy + resource**: host-derived. Effective value = the decoded config, which the host obtains by decoding the entry's composed options against the plugin's schema (the `Plugin` value gets a `readonly config?: Schema` and `readonly settings?: ReadonlyArray<Knob>` alongside `inject`, read off `BundleModules.read` for built rows and off the mounted value for definitions). `setBy` from the per-key map; `default` when no door supplied it.
- **vault**: plugin-reported, live, through a keyed service the way `Wakes` and `Kinds` are (`services.ts:1525`): `Settings.report(readings)` stamped with the fiber name, replaced on each call, withdrawn by the activation's finalizer. Kolu calls it from the revision handler it already has (`kolu/src/server.ts:796-807`); xyne-spaces from its revision handler (`xyne-spaces/src/server.ts:394-404`). A row that is off reports nothing, so its vault chips leave with it, which is the "absence rather than a disabled copy" rule (`docs/running.md:390`).
- **memory**: host-derived from the `localStates` map (`services.ts:1659-1668`): one reading `{ key: "remembers", kind: "memory", door: <path>, setBy: "default" }` for a row that minted the door.

Secrets: the reading is built without `value` at the composition root, and `packages/surface`'s schema test pins that a `secret: true` reading has no `value` key. The wire, not the panel, is the fence.

## 3. What the plugins panel shows, links to, and refuses

The panel is `⧉`; it stays the instance's, grouped as #543 left it. Its job grows from "enablement plus whatever landed in `config:`" to "**every declared knob of every row, with its door and its author**", and nothing more.

### 3.1 Shows

Per row, beside the name, one chip per reading that is **in force**: `key value` with an author mark when the author is not the default (`·flag`, `·env`, `·vault`, `·bundle`). The rule for which readings are chips:

- every `policy` reading (the serve's behaviour is always drawn, which is #543's point, now without a YAML copy);
- every `resource` or `vault` reading whose `setBy` is not `default`/`unset`;
- every `secret` reading, as `set` or `unset`, never a value;
- the rest under a per-row disclosure, `▸ N at their defaults`, listing key, door and default. This keeps identity at defaults from being five chips on a row the human already called "portrait spammy" (`rows.ts:29-49`), while still answering "which variable would I set".

Enablement stays the switch, the state word and the foot line (`pluginsStarted`), unchanged.

### 3.2 Links to

- A `vault` reading with a `link` draws the chip as a file link to that served path (the kolu wrench, generalised; `Feed.tsx:55-116` keeps its own wrench). A vault reading with no file (no `Kolu.olai`) has no link and sits under the disclosure with its default, the way `NO_KNOBS` draws no foot.
- A `memory` reading draws the path as text, not a link (nothing serves `$XDG_STATE_HOME`).
- The row name links to `docs/plugins/<name>.md`, which every plugin has (`packages/tests/plugin_docs.test.ts`). That is where the long account of a knob lives; `says` is one sentence.

### 3.3 Refuses

- Any control that writes a durable setting. The panel writes nothing to YAML, env, nix or the vault; the one write it has stays `plugins.approve`. A knob is changed where it is authored, and the chip's door says where.
- Secret values. Set/unset only, fenced on the wire (§2.3).
- Browser preferences. They belong to `⚙` (`docs/running.md:67`), on purpose, and they never travel on the roster.
- Machine memory contents. Path only.
- `_olai/Properties.olai` as a knob. It is the vault's vocabulary, core's to read (`packages/format/src/typing.ts:91-96`, "DATA, NOT CONFIG"), and a plugin's only relation to it is the kind it *claims*. What a row may say is already said by `Kinds`; drawing the vault's declarations on a plugin row would be the panel reading somebody's outline.
- Any plugin name. The walk is unchanged; every word on a chip arrives as data. `fence.test.ts` "general production packages name no plugins" keeps holding over `plugin-inspector`.

### 3.4 The foot: this serve's own knobs (severable)

Bind address, hostname, log level, log face, allowed origins and whether a bearer is required are the serve's, not a row's. They get the same reading shape on the roster (`PluginRoster.serve: Array<reading>`) and one collapsed group at the foot, **This serve**, under the "Started with" line. It answers the other half of the issue's question, "what is this serve running, and how is it set", without inventing a row. Severable into a later PR because no rule in §6 depends on it.

## 4. Where defaults live

**One home: the plugin's `Config` schema field, as a decoding default.** Every other surface derives from or is pinned to it.

Why the schema and not the YAML row, which is what #543 chose for git:

1. A vault-defined plugin has no YAML row (`rows.ts:465-470`), so YAML cannot be the home for every plugin, and a home that covers only built rows is two homes.
2. A plugin may not import the registry (`fence.test.ts` "a plugin does not import the registry"), so a plugin's own decode cannot read a default that lives in `olai.yml`. Today git works around this by reading `@olai/format`'s constant for the default and the YAML for the panel, which is the drift #543's `rows.test.ts:74` test holds together by equality. A test that holds two copies equal is the shape this repo calls a monument to duplication (`rows.ts:5-17`).
3. Cordis's own answer is the schema (`plugin.ts:197-199`, "Defaults live on the fields"). The loader entry carries what was *set*; the activation folds defaults. The panel's defect was drawing the entry instead of the effective value, and §2.3 fixes that directly.

So:

- `olai-plugin-git`'s `Config` (`git/src/server.ts:51-55`) gives `commit` and `push` decoding defaults of `COMMIT_DEFAULT` / `PUSH_DEFAULT` from `@olai/format`. That floor package stays the constant's spelling; the schema is the home in the sense that every reader goes through it.
- `olai-plugin-vault`'s `Config` (`vault/src/format.ts`) defaults `format` to `"olai"`.
- `olai.yml` drops `config:` from the git and vault rows. `packages/bundle/olai.yml:26-32` and `:75-87` are rewritten to say what `config:` is: a build's overlay, absent in the ordinary case.
- **Fence**: `rows.test.ts` replaces "the git row's config is the built-in default" with its inverse: for every row with `config:`, no key equals the schema's decoded default. A copy is refused at the seam where it would be written.
- `--help`: `gitPolicy.ts` keeps its prose and its imports; `gitPolicy.test.ts` already pins that the sentences name every mode and the default. One new test in the git plugin pins `decode(Config)({})` equals `{ commit: COMMIT_DEFAULT, push: PUSH_DEFAULT }`, so `--help`, the schema and the constants are one chain with a test on each link.
- nix: a text test (`packages/server/src/nixModule.test.ts`, reading `nix/home/module.nix`) asserts each `services.olai` option that projects a flag names the same default the flag's sentence names (`manual`, `off`, `Math.round(QUIET_MS/1000)` seconds, `7714` is the module's own and stays), and that every option maps to a flag `web` declares or to an env door a row declares. `nix/home/check.nix` keeps proving argv.
- docs: `plugin_docs.test.ts` grows one claim: every `resource` door a built row declares is named in `docs/running.md`. That is what would have caught `OLAI_SPACES_URL`/`OLAI_SPACES_TOKEN` being absent from the operator page today.

## 5. Is `config:` the right overlay target, and which env knobs stay env

**Yes, for policy.** A flag or nix option is a patch onto the row's config, exactly as now. Nothing about `--commit`, `--push`, `--plugins` and the nix projection changes except that the patch now records which keys it set.

**Resources stay env, for reasons that still hold, and the reasons are now checkable:**

1. A resource names what this *machine* has (a path, a socket, a header name, an origin); it is set where the machine is wired (the unit, the wrapper's `--set-default`, `environmentFile`). A flag would be a second author for one deployment fact (`olai.yml:103-106`, `docs/plugins/identity.md:28`).
2. Subprocesses inherit env, not argv: the agents olai spawns read the same environment (`module.nix:208-215`), and `OLAI_ODU_BIN` is spliced onto PATH by the wrapper before TypeScript runs (`default.nix:204-205`).
3. Secrets may not be argv (`ps`), and `environmentFile` exists precisely so they are not in the store (`module.nix:199-200`).
4. `--help` is for policy a person reads without knowing it exists (`pluginPolicy.ts:18-23`). A page listing twelve `OLAI_*` names would bury the six flags.

What changes is that resources **grow a declaration and a reading**, not a `config:` line in YAML or a flag. In Cordis terms they do become entry options (the host feeds them into the row's config from env), which is what lets one decode, one reading and one panel cover them; in the operator's terms nothing moves.

Three specific rulings:

- **`OLAI_ACP_AGENT=""` turning the whole chat off is argued down.** Today the chat row reads it as a panel-wide off switch before probing (`chat/src/agents/roster.ts:205`) while the claude row reads it as its adapter (`claude/src/server.ts:65`). Under the taxonomy it is the claude row's `resource`; empty means *this engine has no adapter here*, which the chat panel already draws as an engine that is not installed (`NoAgent.tsx`). "Not this time" is `--without-plugins=chat` (added after that sentence in `docs/running.md:325` was written) or the switch. One meaning per variable is the whole point of a map; the second meaning is retired. This is an open question for the human (§8) because it changes documented behaviour and the e2e testlib pins the empty value (`serve.testlib.ts:36`).
- **`OLAI_CHAT_IDLE_MS` is a policy in env clothing** (how long an idle conversation is kept, `chat/src/server.ts:199`). It is not a machine fact. Either it becomes a `policy` knob with no flag (bundle/default only, drawn on the panel) or it grows a flag. Recommendation: `policy`, no flag, declared, so it is at least visible; a flag is a later decision.
- **`OLAI_LOG_LEVEL` stays env** with its documented reason (a unit cannot pass a flag without rewriting argv, `docs/running.md:196`) and its nix projection. It is the serve's, drawn in the foot (§3.4), not a row's.

## 6. How a vault-defined plugin declares knobs without a seventh door

Same declaration, same schema, same reading. A definition already uses `definePlugin` (`vault-plugins/src/runtime.ts:293`, `loading.ts:37`), so `config` and `settings` are available to it today; what is missing is that `OwnedLoader.mount` passes no config (`loading.ts:11-13`), so any required field fails the row (`plugin.ts:246-250`).

Values come from doors that already exist:

- **`policy` fields: properties on the definition node itself.** The node already carries `plugin` and `approved` (`source.ts:67-68`); a `policy` key becomes a third property, `custom[key]`, a string the schema decodes. That is the vault door, and it is the strongest case for it: a directory saying how a plugin *the directory itself defines* behaves. It travels with the vault, is in the ledger beside the source, and is written through the ordinary write door with the ordinary subtree fence. Editing a config property does **not** change `versionOf` (`source.ts:241` hashes only the two halves), so it does not re-ask for approval: a person approves code, not settings. `dynamic-plugins.md:117-119` ("the source is the configuration") is rewritten to say so.
- **`resource` fields: env**, fed by the host exactly as for built rows. A definition that needs `OLAI_MY_TOKEN` declares it `secret`, and the panel shows `set`/`unset`.
- **`vault` fields**: the plugin reads its own file and reports, as kolu does.
- **Enablement** is unchanged: `approved:` plus the switch; a definition's properties cannot switch a host tool (constraint 2 keeps holding, because `disabled` is not a `Config` field of anything).

Fence: `plugin` and `approved` are refused as `Config` keys of a definition (they are the loader's words); `definedIn` (`source.ts:126`) is what reads the node, so it is where the properties are collected and where the refusal lands, on the row's `fault`, next to the existing "bad word" faults (`source.ts:184-203`). The `plugins.inspect` answer (`vault-plugins/src/surface.ts:120-150`) grows `layout.config: "a property per Config key on the plugin node"` so an agent writing a definition learns it from the thing that enforces it.

`plugins.run`'s answer (`server.ts:85-109`) may carry the row's `settings` so an agent can read what it wrote back. That is a read, on the face an agent already has; no write is added.

## 7. The constraints: kept or argued down

| Constraint (issue) | Verdict | How the design keeps it, or why not |
|---|---|---|
| Instance vs browser | **kept** | `browser` is not a `KnobKind`; nothing browser-owned travels on the roster. The panel test "the panel names no preference" pins that no chip key equals a `preferences.sections` key. |
| Vault vs host | **kept, and made mechanical** | Only a definition's own node feeds `policy` fields; a built row's config is composed from bundle, flag and env only, never from the vault. `disabled` is never a `Config` field. A `vault` knob is plugin-reported and read-only on the panel. |
| Secrets stay out of the vault | **kept** | `secret` implies `resource` implies env door; a reading for a secret is built without `value`; a definition may declare a secret only as `resource`. |
| Env names a resource; a flag names a policy | **kept, and made mechanical** | `policy` knobs have no `env` door, `resource` knobs have no `flag` door, checked by `settings.test.ts` over every built row. One documented exception, `OLAI_LOG_LEVEL`, is the serve's, not a row's. |
| No settings file that survives restart for enablement | **kept, and widened** | Nothing on the panel writes anywhere; no `policy` knob has a machine-local door. `memory` is a door the panel names and never opens. |
| `olai.yml` is the only place a plugin is named; the inspector hardcodes nothing | **kept** | Declarations live in the plugin; readings are data; the panel walks. `fence.test.ts` unchanged and still green over the new files. |
| Panel flips are session-only; an agent cannot flip over MCP | **kept** | Untouched. `plugins.run` gains a read of settings, no write. |
| Git defaults on the YAML row (#543) | **argued down** | §4: YAML cannot be the home for a definition and cannot be read by the plugin's own decode; the schema can be both. The panel draws effective config, so nothing #543 fixed regresses; the e2e `pinned_git_policy.feature` steps keep passing on the same chips. |
| `OLAI_ACP_AGENT=""` turns chat off | **argued down, pending the human** | §5. One variable, one meaning; "not this time" has a flag now. |
| Kolu's values stay server-side, only the filename crosses (`kolu.ts:210`) | **relaxed** | The three durations are a `vault` reading on the roster, as strings in the file's own grammar (`60s`, `10m/4`). Nothing secret or heavy crosses; the wrench keeps opening the file. |

## 8. Open questions for the human

1. **Reverse #543's YAML defaults** (git and vault rows lose `config:`; defaults on the schema; a test refuses a YAML copy)? Recommended yes, §4. The alternative is keeping YAML as the home for built rows and accepting that definitions are different.
2. **Retire the empty-`OLAI_ACP_AGENT`-turns-chat-off meaning** in favour of `--without-plugins=chat`? Recommended yes, §5. It changes `docs/running.md:325`, `docs/chat.md`, `NoAgent.tsx`'s `switched-off` arm and the e2e testlib default.
3. **Env values into the composed config** (host reads env into a row's options; plugins stop reading `Env.vars`)? Recommended yes, §2.2; it is what makes one reading possible. The alternative is plugins reporting resource readings themselves through `Settings.report`, which keeps `Env` reads in plugins and makes "who set it" a claim each plugin repeats.
4. **Chip rule**: policy always, resource/vault/secret only when set, the rest under `▸ N at their defaults` (§3.1)? Or every declared knob as a chip?
5. **`OLAI_CHAT_IDLE_MS`**: declare as policy-without-flag (recommended), grow `--chat-idle`, or leave undeclared.
6. **This serve** foot (§3.4) in the first PR or later?
7. Kolu's durations crossing the wire (§7 last row): acceptable, or keep only the filename?

## 9. Mechanical rules a reviewer can check

Each rule names the test that pins it. A PR that claims to implement a step must add or move the named test.

| # | Rule | Test |
|---|---|---|
| R1 | Every `policy`/`resource` knob key is a `Config` field and vice versa; `vault` knobs are not `Config` fields. | `packages/bundle/src/settings.test.ts` (imports every row's module, like `declaredKinds`) |
| R2 | `policy` knobs have no `env` door; `resource` knobs have no `flag` door; `secret` implies `resource`. | `settings.test.ts` |
| R3 | No `olai.yml` `config:` value equals the schema default for that key. | `packages/bundle/src/rows.test.ts` (replaces `:74`) |
| R4 | `decode(Config)({})` equals the floor constants for git; `format` defaults to `olai`. | `packages/plugins/git/src/server.test.ts`, `vault/src/format.test.ts` |
| R5 | A `secret` reading on the wire has no `value`. | `packages/surface/src/plugins.test.ts` |
| R6 | `setBy` is `flag` exactly for the keys a flag set, `bundle` for YAML, `env` for a set variable, `default` otherwise; `--commit=manual` typed reads `flag`. | `packages/server/src/runtime.test.ts` |
| R7 | A row switched off publishes no `vault` readings; switched on, they return on the next revision. | `packages/plugins/kolu/src/server.test.ts` + e2e |
| R8 | The panel draws a chip per in-force reading with an author mark, and the disclosure for the rest; no chip key names a preference. | `packages/plugins/plugin-inspector/src/rows.test.ts` |
| R9 | Every `resource` door a built row declares is named in `docs/running.md`; every plugin's `docs.md` names its own doors. | `packages/tests/plugin_docs.test.ts` |
| R10 | Each `services.olai` option projects a flag `web` declares or an env door a row declares, and names the same default. | `packages/server/src/nixModule.test.ts` |
| R11 | A definition's `Config` may not use `plugin` or `approved`; a config property edit changes no version and needs no re-approval. | `packages/plugins/vault-plugins/src/source.test.ts`, `runtime.test.ts` |
| R12 | `--help` for `--commit`/`--push` names the modes and the default (existing). | `gitPolicy.test.ts` |
| R13 | General packages still name no plugin; the inspector's new files are on the fence. | `fence.test.ts` (existing, must stay green) |

## 10. Implementation plan for Codex

Ordered. Steps 1 to 5 are one PR (the map, the reading, the panel, the home for defaults); 6 to 9 are severable, each its own PR. Every step lists files, the tests that pin it (numbers from §9), and docs, which land in the same PR as the code.

**Step 1: the vocabulary.**
Files: `packages/plugin-api/src/contract.ts` (`Knob`, `KnobDoor`, `KnobKind`, `KnobReading`), `packages/plugin-api/src/services.ts` (`Settings` keyed service: `report(readings)`, provided in `openPlugins` beside `Wakes`, withdrawn with the fiber; export it from the server door), `packages/effect-cordis/src/plugin.ts` (`definePlugin` takes `settings?`, and the returned `Plugin` carries `config` and `settings`), `packages/surface/src/plugins.ts` (`settings` on `BuiltPlugin`, `config` removed).
Tests: R5; `services.test.ts` for the keyed service's stamp and unwind (mirror the `Wakes` cases).
Docs: `docs/internal/plugin-system.md` gets a "settings" section beside kinds and wakes.

**Step 2: defaults go home.**
Files: `packages/plugins/git/src/server.ts` (decoding defaults from `@olai/format`), `packages/plugins/vault/src/format.ts` (`format` default), `packages/bundle/olai.yml` (drop `config:` on git and vault; rewrite the `config` paragraph at `:26-32` and the git row comment at `:75-87`), `packages/bundle/src/rows.test.ts`, `packages/bundle/generate.ts` (no change needed; `configOf` still admits an overlay).
Tests: R3, R4, R12 (unchanged, must still pass).
Docs: `docs/running.md:277` ("The git row's `config:` in `olai.yml` is that default") and `docs/plugins/git.md:24` say the default is the plugin's and YAML is an overlay.

**Step 3: provenance at the composition root.**
Files: `packages/server/src/gitPolicy.ts` (`gitConfigPatch` returns per-key `setBy`; keeps writing the whole object because Cordis copies the field), `packages/bundle/src/bundle.ts` (`mountBundle` composes bundle → flag → env per row using each module's `settings`, records `setBy` per key; `configsOf` becomes `settingsOf(host)` returning effective decoded values plus `setBy`), `packages/effect-cordis/src/loader.ts` (`rowConfigs` stays as the raw read), `packages/server/src/serve.ts` (passes `vars` into the composition; `settings: () => settingsOf(...)`), `packages/server/src/runtime.ts` (`rosterOf` joins host readings, `Settings` reports and the `memory` line).
Tests: R6; `bundle/src/composition.test.ts` for the env feed with a fake `vars`.

**Step 4: the panel.**
Files: `packages/plugins/plugin-inspector/src/rows.ts` (`pluginConfig` becomes `pluginChips` + `pluginDefaults` implementing §3.1), `Panel.tsx` (chip with author mark, file link for `vault` readings via the app's `FileLink`, the `▸ N at their defaults` disclosure, secret as `set`/`unset`), `testids.ts` (`pluginChip`, `pluginChipAuthor`, `pluginDefaults`), `packages/tests/step_definitions/preferences_steps.ts` (the existing "configured X as Y" step reads the chip; add "set by", "at its default", "names the door").
Tests: R8; e2e `packages/tests/features/settings_doors.feature`: git at defaults draws both chips with no author mark; `@pin:commit=auto` draws `·flag`; `xyne-spaces` on with no env draws `OLAI_SPACES_TOKEN unset`; a secret chip never shows a value; the disclosure lists identity's five doors on a serve with none set.
Docs: `packages/plugins/plugin-inspector/docs.md`, `docs/running.md` "Which integrations this serve runs" gets **The map** (the §1 table in prose) and the chip rule.

**Step 5: resources declared and fed (the env move).**
One plugin per commit so each is reviewable: `identity` (`who/config.ts` becomes the schema; `server.ts:54-57` reads `config`), `claude`/`codex`/`pi` (`adapter` field from `OLAI_ACP_*`), `chat` (`searchPath` from `OLAI_AGENT_PATH`, `idle` as policy), `odu` (`bin` from `OLAI_ODU_BIN`), `kolu` (`socket` from `PADI_SOCKET`; `socket.ts:45` reads config), `xyne-spaces` (`url`, `token` secret). Each plugin's `docs.md` names its doors.
Tests: R1, R2, R9 (extend `plugin_docs.test.ts`); each plugin's existing env tests move from `vars` to `config`.
Docs: `docs/running.md` gains the Spaces variables; `:128` corrected for the login header; `docs/index.md:28-43` pointers now true.

**Step 6 (severable): vault readings.**
Files: `packages/plugins/kolu/src/server.ts` (report `held-for`/`nag`/`heartbeat` with `link: file` from the revision handler), `packages/plugins/xyne-spaces/src/server.ts` (report `xyne-channel` as a bound count), `kolu/docs.md`, `xyne-spaces/docs.md`.
Tests: R7; e2e: editing `_olai/Kolu.olai` moves the chip without a reload (extend `the_feed_opens_its_config.feature`).

**Step 7 (severable): definitions take config.**
Files: `packages/plugin-api/src/loading.ts` (`OwnedLoader.mount(plugin, config?)`), `packages/plugins/vault-plugins/src/source.ts` (collect `custom` keys as config; refuse `plugin`/`approved` as keys), `runtime.ts` (pass config; the version stays the two halves), `surface.ts` + `server.ts` (`inspect.layout.config`, `run` returns `settings`), `docs/dynamic-plugins.md`.
Tests: R11; e2e in `a_plugin_the_vault_defines.feature`: a definition with a `Config` field reads the property, an edit re-applies without a fresh approval, a reserved key faults the row.

**Step 8 (severable): This serve foot and the memory line.**
Files: `packages/surface/src/plugins.ts` (`serve` readings on the roster), `packages/server/src/serve.ts`/`runtime.ts`, `Panel.tsx`, `packages/server/src/allowedOrigins.ts`/`hostname.ts`/`@olai/log` (expose what they read as readings; no behaviour change).
Tests: `rows.test.ts` foot group; `runtime.test.ts` for the `memory` line only on rows that minted a door.

**Step 9 (severable, needs the human's yes on Q2): one meaning for `OLAI_ACP_AGENT`.**
Files: `packages/plugins/chat/src/agents/roster.ts:205` (remove the `switched-off` arm), `chat/src/wire/members.ts:1736-1748`, `chat/src/browser/chat/NoAgent.tsx`, `packages/server/src/serve.testlib.ts:36`, `docs/running.md:325`, `docs/chat.md:34-44`.
Tests: chat's roster tests; e2e `choosing_an_agent.feature` gains "an empty adapter is an engine that is not here, and the panel stays".

**Doc drifts found on the way, to fix in the same PR as step 5** (from the surveys): `docs/running.md:128` (login header cannot be emptied); `docs/running.md` lacks `OLAI_SPACES_*` and the "empty by default" of `OLAI_ALLOWED_ORIGINS`; `docs/index.md:28,29,31,43` point at plugin pages that do not mention the variables; `docs/running.md:476` and `main.ts:260` tell an operator to set `$OLAI_TOKEN` for a bearer the server mints per process and never reads (`serve.ts:87`, `mcpClient.ts:44`); `packages/state/README.md` and `docs/architecture.md:153` describe a LocalState migration that no longer exists.

## 11. What this does not do

- No settings file, no `--dump-config`, no CLI verb against a running serve, no edit of any door from the panel.
- No sandbox for definitions; the boundary stays approval by a person.
- No change to how `--plugins`, profiles and the switch decide enablement.
- No new generated file in `packages/bundle`; declarations are read off the modules the bundle already imports (`declaredKinds`, `bundle.ts:171-182`), so the fence is unchanged.
