# File kinds as plugins

A handoff for the agent implementing branch `file-kinds`. One PR. Read
`CLAUDE.md`, `docs/architecture/cordis.md` and `docs/architecture/plugin-system.md`
first; run `just cordis-graph` and focus `vault`, `files`, `markdown`, `navigation`
to see the graph this PR changes.

## Goal

Every kind of file olai serves is claimed by a plugin row, not by a table in
core. Today `packages/format/src/kinds.ts` claims six kinds, the `files` row
draws all six icons and nouns, the `markdown` row draws all five body faces, and
the vault's media route reads the table's `fetched` column. After this PR:

| row | status | server half registers | browser half contributes |
|---|---|---|---|
| `olai` | new | `.olai`, `holds: nodes`, JSONL `parse` and `serialize` | nothing; `outlines` draws every `holds: nodes` kind |
| `markdown` | existing | `.md`, `holds: text`, `kept` | document face, editor, glyph, noun |
| `hypertext` | new | `.html`, `text`, unkept, fetched | sealed-frame face, glyph, noun |
| `csv` | new | `.csv`, `text`, unkept | table face over `vault.files.body`, glyph, noun |
| `image` | new | nine picture suffixes, `bytes`, fetched | `<img>` face, glyph, noun |
| `pdf` | new | `.pdf`, `bytes`, fetched | `<object>` face, glyph, noun |

`org` is NOT in this PR and no code may mention it. It becomes a row like `olai`
later. PR #466 is the reference for what an org row will need from the contract;
read its `docs/org2-poc.md` for the list, then close it.

## Rulings (do not reopen)

1. **One registry**, key `vault.file-kinds`. A claim carries `holds`, `kept`,
   `fetched`, `noun`, `article`, and when `holds: "nodes"` also `parse` and
   `serialize`. Not two registries.
2. **Owned by the vault row**, minted in `vault-setup` exactly as `VaultViews`
   is (`packages/plugins/vault/src/views.ts`, `setup.ts`). Not host-provided,
   not files-owned. Files needs `ops`, which vault offers; vault must have the
   table before it opens the store; a host registry would hide the owner.
3. **Browser registries belong to their readers.** `files.kinds` (glyph, noun,
   article, test id) is a location owned by `files`, beside `files.types`.
   `navigation.pages` (the face a kind's page draws) is a location owned by
   `navigation`. Kind rows contribute to both on their browser half.
4. **Unknown suffix means unclaimed file.** No static list of suffixes in core.
   A `.org` on a serve with no org row is not in the set, exactly like `.txt`.
5. **`olai` can be switched off.** Off: every `.olai` is unclaimed and a bare
   path can only be told `no row claims `.olai``, because the registry holds
   live claims and nothing maps a suffix to a row that is not there. The row
   IS named where its id is already in hand: the vault's `format` config. So
   mints refuse naming it, and the surfaces that already know a convention
   and read the configured outline row off the `file-kinds` cell (below) say
   `the olai row is off`: the `/trash` and `/agenda` pages, which have routes
   of their own, and the Inbox and Pins SIDEBAR entries, which are where those
   two conventions live. Ruled: no new routes for Inbox or Pins. Their files
   open at ordinary addresses, and an ordinary address with no claim is the
   "nothing by that name" page saying `no row claims `.olai``, like any other.
   No roster lookup and no suffix-to-row table anywhere.
   The plugins panel gets a `switchHint` saying so. Honest beats safe here.

   **Its switch is session-only, like the vault's and the settings reader's.**
   `_olai/Settings.olai` is itself an outline: a durable `on: no` for the row
   that claims its suffix would unclaim the file that says so, the reading
   would fall to defaults, and the row would come back. The host already has
   the door for exactly this (`packages/server/src/configuration.ts`,
   `sessionOwners`): a reader owner's `on` in the file is ignored with a
   warning, its switch is session-only and wears the dashed ring, and applied
   patches for every other row stand while the reader is withdrawn. Ruled:
   the set of reader owners gains a third member, DERIVED like the other two
   and never spelled: the row whose registered claim holds the settings file's
   suffix, read off `FileKinds.current()` (its `kind` is the row id). Today
   that is `olai`; with an org row and an `.org` settings file it would be
   `org`, with no code change. Turning `olai` off therefore loses the settings
   reading until it is on again, exactly as turning the vault off does, and
   other rows keep their applied patches meanwhile.
6. **The `FileKind` closed union goes.** Kind names are strings. The three
   `Record<FileKind, …>` tables in `files/src/contracts/icons.tsx`,
   `files/src/contracts/kinds.ts` and `markdown/src/browser/document/faces.tsx`
   dissolve into per-row contributions. Compile-time "every table names every
   kind" is replaced by registration and a runtime "no row claims this" page.
7. **Vault `format` config** becomes the id of the row new outlines are minted
   in. Default `olai`. A mint whose row is off refuses naming it.
8. **Conventions match by stem.** `Trash`, `Inbox`, `Pins`, `Properties` in
   `_olai/` with any `holds: nodes` suffix. Two such files with one stem is a
   new set finding, `ambiguous-convention`, naming both.

## The contract (`@olai/plugin-api`)

```ts
// packages/plugin-api/src/services.ts, beside VaultViews
export interface FileClaim {
  readonly exts: readonly [string, ...string[]] // matched exactly, first is what a mint writes
  readonly holds: "nodes" | "text" | "bytes"
  readonly kept: boolean
  readonly fetched: boolean
  readonly noun: string                       // "outline", "page"
  readonly article: "a" | "an"
  readonly format?: OutlineFormat             // required iff holds === "nodes"
}
/** What the table holds: the claim plus WHO made it, stamped by the registry
 *  off the registering fiber's binding (`moduleOwner`), never supplied by the
 *  row. `kind` IS the row id. One row, one claim. */
export interface ComposedClaim extends FileClaim { readonly kind: string }
export interface FileKinds {
  readonly current: () => ReadonlyMap<string, ComposedClaim>   // by kind, i.e. by row id
  readonly changes: Stream.Stream<void>
  /** Claim, for as long as the calling plugin is loaded. Checked and installed
   *  in one synchronous step over every suffix of the claim: a suffix already
   *  claimed, a suffix that ends in one already claimed, or a second claim
   *  from the same row FAILS the calling fiber and installs nothing; its
   *  finalizer removes only what this call installed. */
  readonly register: (claim: FileClaim) => Effect.Effect<void, never, Scope.Scope>
}
export const FileKinds = serviceTag<FileKinds>("vault.file-kinds")
```

Consequence for readers: nothing compares a kind to the string `"outline"` or
`"document"` any more. Outline-ness is `holds === "nodes"`; a document is
whatever the `markdown` row claims. Glyph and page lookups are keyed by the row
id the registry stamped. This is what keeps a second outline row (`org`) from
needing a word core knows.

`OutlineFormat` lives in `@olai/format` (`packages/format/src/format.ts`):

```ts
export interface OutlineFormat {
  /** `claims` is the table in force at the call, handed in by whoever holds it
   *  (the codec, the writer's read-back, the vault's `outlineDiff`). A format
   *  is a pure function of its three arguments; it never reads the registry
   *  it was registered into. `outlineDocument` needs claims to derive a
   *  file's links and title, which is why the argument exists. */
  readonly parse: (file: string, contents: string, claims: Claims) => Result.Result<Outline, ReadonlyArray<OutlineError>>
  readonly serialize: (nodes: ReadonlyArray<Node>) => string
}
```

Ruled: three arguments, not a wrapper closing over the service. A row reading
`FileKinds.current()` inside the function it registers there would be reading
its own registration, and a per-call read is still a hidden dependency. The
codec already holds the table; it passes it.

`@olai/format` also gains an inert value type the rest of the leaf reads
instead of the table:

```ts
// packages/format/src/kinds.ts, replacing FILE_KINDS
export interface Claims { readonly byKind: ReadonlyMap<string, Claim>; readonly byExt: ReadonlyMap<string, string> }
export const claims = (list: Iterable<Claim>): Claims            // builds both maps, refuses a suffix ending in another
export const NO_CLAIMS: Claims
export const fileKind  = (claims: Claims, path: string): string | null
export const bareOf    = (claims: Claims, path: string): string
export const stemOf    = (claims: Claims, path: string): string
export const holdsBody = (claims: Claims, kind: string): boolean
export const holdsText = (claims: Claims, kind: string): boolean
export const bodyKind  = (claims: Claims, path: string): string | null
export const textKind  = (claims: Claims, path: string): string | null
export const unkept    = (claims: Claims, path: string): boolean
export const isFetched = (claims: Claims, path: string): boolean
export const mintExt   = (claims: Claims, kind: string): string | null   // replaces OUTLINE_EXT, DOCUMENT_EXT
export const parserFor = (claims: Claims, path: string): OutlineFormat | null
```

`Claim` is the existing interface plus `kind`, `noun`, `article`, `format?`.
This is the `KindVocabulary` pattern: a value handed down, never a module-level
table. `Claims` must be a plain value so it can cross the wire minus `format`.

## Server plumbing

- `vault-setup` (`packages/plugins/vault/src/setup.ts`): `openViews()` gains
  the file-kind table, same double-claim refusal wording as ledger and search.
  `VaultSettings` gains `claims: { get current() }`, a getter like `kinds.enabled`.
  Offer `FileKinds` from the same door.
- `vault` main (`server.ts`): `codecFor(settings.kinds, settings.claims)`.
  `codecs` table and `FORMATS` in `format.ts` go; `Config.format` stays as a
  row id (`Schema.String`, default `"olai"`, description updated).
- `vault-revalidation`: also listens to `FileKinds.changes`. A row that mounts
  after the store opened brings its files in on the next look; one that leaves
  takes them out. Unclaimed paths are not stamped, so no cache invalidation is
  needed beyond the re-probe.
- Host `followConfiguration` (`packages/server/src/configuration.ts`): the
  `sessionOwners()` derivation gains the row whose claim holds the selected
  settings file's suffix, asked of the vault's `FileKinds` service when it is
  offered (absent vault means no such owner, which is already the case). The
  existing warning wording (`this reader's switch is session-only so the file
  cannot disable its own reader`) covers it unchanged.
- `@olai/ops` `codec.ts`: `match` asks `fileKind(claims.current, path) !== null`.
  `decode` is `parserFor(claims, path)?.parse(path, contents, claims)`, the
  one `claims` value read at the top of the call. `byName` through `unkept`. All via
  the getter, read per call.
- `@olai/ops` writer (`ops.ts:668`, `following.ts:138`): serialize through
  `parserFor(claims, planned.file).serialize(nodes)` and read back through
  `.parse(planned.file, text, claims)`, the same `claims` value the planner was
  judged against. A planned file whose suffix has no format is a defect
  refusal, same shape as the existing read-back guard.
- `@olai/ops` `plan.ts` `outlinePath`: takes `mintExt(claims, config.format)`,
  the row id the vault's `format` config names, and checks that claim
  `holds === "nodes"`; when no claim is registered under that id, refuse with
  `the olai row is off, so no outline can be created` (the id comes from the
  config, not from any suffix). `markdown_create` mints
  through `mintExt(claims, "markdown")`: the markdown row's own verb may name
  its own row id, since it is the one thing that row knows about itself.
- `@olai/format` `node.ts` conventions: `TRASH`, `INBOX`, `PINS`, `PROPERTIES`
  become stems; `outlineCalled(files, stem)` matches by `stemOf` over
  `holds: nodes` files. `TRASH_FILE` and `isTrashed` take claims. The
  `ambiguous-convention` finding lives in `rules.ts`.
- `git` (`packages/plugins/git/src/ledger/committed.ts:274`): parse the
  committed side through the format for that path, reached through `Ops`
  (`Ops` gains `parserFor`). Git's server half adds `Ops` to its `needs`; it
  already needs `Vault`, and vault needs nothing of git, so no cycle. The
  ledger it registers into `VaultViews` is unchanged. `pending.ts` and
  `pending.testlib.ts` follow. Git must not import the `olai` row's module for
  the parser: that would parse a committed `.org` with the JSONL reader the day
  an org row exists.
- `chat` (`packages/plugins/chat/src/browser/chat/outline.ts`): the browser
  must not parse. Add a procedure `outlineDiff({ path, oldText, newText })` on
  the VAULT's file surface (`packages/plugins/vault/src/file-surface.ts`),
  answered with `parserFor(claims, path)?.parse(path, text, claims)` on each
  side and `@olai/format`'s `changesOf`, `claims` being the vault's current table.
  Chat's browser reaches it as `outlineDiff` on the `vault.files` service it
  already declares (`fileAccess`); the vault's browser half calls its own wire
  procedure behind that method, since `Wired` exposes only the calling row's
  own surface. With the vault absent the service is absent and chat's
  reference component is not mounted, which is the existing rule. Chat draws
  the answer. For parsing, nothing in `chat/src/server.ts` or `chat/src/wire.ts`
  changes: another branch (`chat-sidebar-ux`) is rewriting those files, and
  parsing is the vault's business anyway. The ONE permitted touch of
  `chat/src/server.ts` is the doorbell's `World` (next item), which is two
  lines: `claims: snapshot.value.claims` at the call site, and
  `readonly claims: Reading["claims"]` on the local `VaultRevision.value`
  narrowing beside `set` and `derived`. That type is deliberately "as much of
  the revision as this half reads", so naming a third field is its own rule;
  do not replace it with the vault's whole `Reading`, which would declare a
  read of everything.
- `kolu/src/wake.ts`, `odu/src/wake.ts`, and the doorbell check
  (`chat/src/server/doorbell.ts`'s `faultedIn`, `@olai/surface`'s `watchable`,
  the browser picker that filters by the same reading): `wake.kinds` was a
  list of kind NAMES typed `NodeKind`, and a kind name is now a row id, which
  a doorbell has no business naming. What kolu and odu mean is "files whose
  records I can walk". So the declaration is renamed to what it means:
  `wake.walks: "nodes"` (the `holds` value the doorbell can be pointed at;
  `"text"` is legal for a plugin that reads prose, as the `NodeKind` docstring
  allowed). `watchable(claims, wake, file)` is `fileKind(claims, file)` having a
  claim whose `holds === wake.walks`; both ends, serve and picker, ask the same
  predicate of their own claims (server getter, browser cell). The judgement
  belongs to whoever holds the reading, and the reading now carries its
  `claims`, so the doorbell's `World` in `chat/src/server.ts` gains one field,
  `claims: snapshot.value.claims`, threaded through `faultedIn` to `watchable`.
  Ruled: that one-line addition at the call site is allowed; a checker that
  fetched claims for itself from `Ops` or a service would be a hidden
  dependency, and the small merge conflict with `chat-sidebar-ux` is resolved
  by whichever branch lands second. Keep the screenshot defect (2026-09-01)
  in the test: a doorbell declaring `walks: "text"` must not be offered outline
  files, and one declaring `"nodes"` must not be offered a document.
- `packages/plugins/vault/src/http/media.ts`: `isAsset` asks `isFetched(claims, path)`.
- `packages/surface/src/seal.ts` interpolates `FILE_EXTS` into the sealed
  frame's script. It becomes a parameter. The caller is the vault's `/media/`
  handler (`packages/plugins/vault/src/http/media.ts`), which builds the sealed
  page server-side and passes the suffixes of its current `Claims` at each
  request; sealing stays vault-owned. The `hypertext` row's browser face only
  points an iframe at that URL and never calls `seal.ts`; the table row above
  saying "sealed-frame face" means the iframe, not the seal.
- `packages/surface/src/plugins.ts` and `packages/surface/src/attach.ts`:
  `attach.ts` is a DIFFERENT list (what a person may hand an agent) and stays;
  check what `plugins.ts` reads and route it through claims.

## Browser plumbing

- `vault.files` (`packages/plugins/vault/src/browser/state.ts`, contract
  `contract.ts`) gains `outlineDiff(path, oldText, newText)`, a method over the
  vault's own wire procedure; `body(path)`, the read of an UNKEPT text file's
  bytes when somebody opens it, over a new `bodies.get` procedure on the
  vault's file surface that refuses any path whose claim is not
  `holds: "text"` and unkept (a kept body rides its own row's collection, a
  fetched kind rides `/media/`). Ruled: this is the vault's read, not
  markdown's. `markdown/src/server/bodies.ts` moves into the vault's server
  half unchanged in behaviour (read then and there, kept by nobody, refusal on
  the entry), markdown's `documents` collection narrows to `.md`, and the csv
  page reads `vault.files.body`. Serving csv through `/media/` was rejected:
  `documents.ts` argues against handing data to a previewed page, and that
  argument stands. Also `claims: Accessor<Claims>` fed by a new `file-kinds`
  cell on the vault surface, and `kindOf(path)`. The cell carries the table
  minus `format`, plus `outlineRow: string`, the id the vault's `format`
  config names. A page that wants to say `the olai row is off` checks
  `claims().byKind.has(outlineRow)`; that is the only way the browser may name
  a row that is not claiming.
  Every browser caller of `fileKind` goes through it: `navigation/src/routes.ts`,
  `files/src/fileTree.ts`, `files/src/Files.tsx`, `files/src/contracts/completing.ts`,
  `files/src/file/making.ts`, `markdown/src/browser/document-route.ts`,
  `markdown/src/browser.tsx`, `outlines/src/browser.tsx`, `search/src/browser/KindSelector.tsx`,
  `chat/src/browser/chat/ToolFrame.tsx`, `outlines/src/browser/Nothing.tsx`, `trash/src/Entry.tsx`.
- `files` owns location `files.kinds`, contribution
  `{ by: { kind } | { holds }, glyph: () => JSX.Element, noun, article, testid }`,
  keyed by row id OR by `holds`. Lookup for a path: its claim's kind first,
  then its claim's `holds`. Ruled: `outlines` contributes once, by
  `holds: "nodes"`, the outline glyph moved out of files' table; so every
  node-holding kind, `olai` today and `org` later, is drawn by the row that
  draws records, and no format row needs a browser half. A body row
  contributes by its own kind.
  `Files.tsx`, `Rail.tsx` and `fileTree.ts` read it. Every path in `heads` is
  claimed by some row's SERVER half, so the only way a listed path has no glyph
  contribution is that row's BROWSER half being absent (failed to load, or the
  tab has not fetched it yet). For that case draw the plain-file glyph and the
  noun from `claims.byKind.get(kind)?.noun`. A row that is OFF has no files in
  `heads` at all (ruling 4), so the tree never shows them and needs no "row is
  off" drawing. `contracts/icons.tsx` and `contracts/kinds.ts` are deleted;
  each glyph moves to its row.
- `navigation` owns location `navigation.pages`, contribution
  `{ by: { kind } | { holds }, page: (address) => JSX.Element, edits: boolean }`,
  keyed by row id OR by `holds`, same lookup order as `files.kinds`. `outlines`
  contributes its tree page once, by `holds: "nodes"`, from its existing
  `content` component; that is the outline page it already owns, now reached
  through the location instead of a hard-wired route.
  `routes.ts` picks the page by `kindOf(path)`. A path the directory does not
  hold (its row is off, or nobody ever claimed the suffix; the browser cannot
  tell these apart and must not try) is the existing "the directory holds
  nothing by that name" page, which now adds `no row claims `.org`` when the
  claims cell has no suffix for it. A path that IS held but whose kind has no page
  contribution mounted (server half claims, browser half absent) draws a page
  saying which row's browser half is missing. Those are the only two cases.
  `markdown/src/browser/document/faces.tsx` is deleted; `DocumentPage.tsx`
  becomes markdown's own contribution.
- New rows `hypertext`, `csv`, `image`, `pdf` under `packages/plugins/<row>/`:
  `package.json` (`olai-plugin-<row>`, exports `./server`, `./browser`,
  `./testids`), `src/server.ts` (`needs: [FileKinds]`, one `register`),
  `src/browser.tsx` with TWO components, `glyph` (`needs: [files.kinds]`) and,
  for csv, a `page` that reads its body through `vault.files.body` rather than
  markdown's collection, and
  `page` (`needs: [navigation.pages, vault.files]` as the floor). Ruled, and
  general: a moved face keeps every service it already named, declared on the
  `page` component and nowhere wider; the list above is the floor, not a cap.
  So hypertext's page also declares `navigation.file-links`, exactly as the
  face did inside markdown, and a reviewer compares the moved face's needs
  against what it named before the move. Neither component needs
  the other. What that buys is exactly this: files being off costs the kind
  its glyph and nothing else, so its page still opens. Navigation being off
  costs it its page, and, as today, the tree and rail go with navigation
  (`files.sidebar` needs `navigation.state`), so the glyph contribution then
  stands unread. That is the same shape as today and is NOT widened here: a
  tree standing without navigation would need a row press that goes nowhere,
  which is a navigation design question, not a file-kind one. `src/testids.ts`. Move the faces out of
  `markdown/src/browser/document/` (`Hypertext`, `Csv`, `Image`, `Pdf`) and the
  glyph paths out of `files/src/contracts/icons.tsx`. `Image.tsx` currently
  reads `PICTURE_EXTENSIONS`; that list becomes the image row's own.
- `olai` row under `packages/plugins/olai/`: `src/format.ts` is today's
  `packages/format/src/parse.ts` + `write.ts` moved with one change:
  `parseOutline(file, contents, claims)` takes the third argument and passes
  it to `outlineDocument`. Keep their headers; they are the format's argument.
  The module exports `format: OutlineFormat = { parse: parseOutline, serialize: serializeOutline }`,
  pure, no service in scope. `src/server.ts` is `needs: [FileKinds]` and
  registers
  `{ exts: [".olai"], holds: "nodes", kept: true, fetched: false, noun: "outline", article: "an", format }`;
  the registry stamps `kind: "olai"`.
  No browser half: a format row encodes records and draws nothing, because
  `outlines` contributes the glyph and the page for every `holds: "nodes"`
  claim (`files.kinds` and `navigation.pages` below). That is what makes a
  later `org` row a server half and a parser, and nothing else.
  `@olai/format`'s tests that exercised `parseOutline` and
  `serializeOutline` move with them or import the row's pure module (a static
  import of pure functions is allowed by `cordis.md`).
- `packages/bundle/olai.yml`: new section `Files` holding `olai`, `markdown`,
  `hypertext`, `csv`, `image`, `pdf`, in that order. `olai` carries
  `profiles: [surface, test-minimal]` like vault and a `switchHint` that says
  turning it off also withdraws the settings reading until it returns. Its
  switch is drawn with the session-only ring because the host derives it as a
  reader owner; nothing in the yml says so. Run
  `bun packages/bundle/generate.ts` (part of `just install`).

## `@olai/format` internals

Every reader of `FILE_KINDS` inside the leaf takes a `Claims`:
`address.ts`, `dates.ts`, `documents.ts` (`isAsset`, `isPicture`, `bodiedOf`,
`pathedOf`), `document.ts`, `incremental.ts`, `message.ts`, `node.ts`,
`page.ts`, `searching.ts`, `set.ts`. Thread it the way `KindVocabulary` is
threaded: on `Reading` / the validator's context, not as a new parameter on
every pure helper where the reading already carries it. `ops/src/refusals.ts`
and `ops/src/plan.ts` read the same.

`FileKind` the schema (`Schema.Literals`) becomes `Schema.String`. `BodyKind`,
`NodeKind`, `TextKind`, `UnkeptKind`, `UNKEPT_KINDS` are deleted. `page.ts`'s
"the directory holds nothing by that name" reading carries the kind string.

**Wire schemas validate shape, not membership** (`address.ts`). `DocumentPath`
and `AtOutline` today filter on the static table at decode time. A schema is
static and the table is live, so the filter cannot stay there, and a
per-surface schema rebuilt from the current claims is rejected: a schema that
changes meaning while a subscription is open is exactly the drift the wire's
"reconnect is a fresh snapshot" rule exists to avoid. Ruled:

- `DocumentPath` is a relative path with `/` separators, no `..` segment, no
  leading `/`, non-empty. Nothing about suffixes. `AtOutline` stays exactly
  `AtDocument` minus the suffix filter: its discriminator is the inherited
  `kind: "document"` (an ADDRESS shape, not a file kind; leave the wire
  vocabulary alone, including the search producers and tests that spell it).
  The brand says what the caller CLAIMS, not what the directory holds.
  `outlineAt(claims, at)` admits an `AtDocument` as `AtOutline` when the claim
  for its path `holds === "nodes"`.
- Membership is decided where the table is: `claimedOf(claims, path)` and
  `outlineAt(claims, at)` in `address.ts`, returning the branded value or
  `null`, are the only constructors ops and the vault use to admit a path off
  the wire.
- The refusal moves one step later and says more. Every procedure and tool
  that took a `DocumentPath` (`markdown_read`, `markdown_write`,
  `outlines_subtree`'s file arm, `outlines_create`, `outlines_move`'s `file`,
  `edit-intents`'s file verbs, `search_nodes`'s file filter) refuses in
  `ops/src/refusals.ts`'s existing `notFound` shape with the sentence
  `` `notes.org` is not a file this directory serves: no row claims `.org` ``.
  That is the ONLY sentence for an unclaimed path: with the `olai` row off the
  registry has no `.olai` entry, so `plan.olai` gets the same sentence with
  `.olai` in it, and nothing may keep a suffix-to-row table to say more. The
  `didYouMean` neighbour list is unchanged. Where a verb needs an outline specifically
  (`outlines_create` on a `.md`), the refusal is
  `` `notes.md` is a document; this verb takes an outline ``.
- The MCP tool schemas advertise `DocumentPath` as a plain relative path; the
  tool description says which kinds the verb accepts, in words.
- e2e: the scenario that typed an unclaimed suffix into the new-file box or a
  tool and expected a schema decode failure now expects the `notFound`
  sentence above. Check `packages/tests/features/` for `is not a field` /
  `expected a relative path` assertions and retarget them.

## Tests

- `packages/tests/kinds.test.ts`: replace both sweeps. New claim: a registered
  suffix is spelled as a string literal only in the row that registers it
  (derive the allowed file list from `packages/plugins/*/src/server.ts`
  registrations). Keep the `attach.ts` and fake-agent exemptions and their
  reasons. Delete the `Record<FileKind>` inventory sweep.
- `packages/tests/extension.test.ts`: unchanged in intent; check it still
  reads the retired spelling nowhere.
- Format round trip: a property test per registered format
  (`parse(serialize(nodes))` is identity, canonical bytes are stable), run
  through the registry so a future row inherits it.
- `packages/bundle/src/fence.test.ts`: the tenancy claims must still hold; the
  new rows import only `@olai/plugin-api`, `@olai/format` and their own package.
- e2e (`packages/tests/features/`): `olai` switched off from the panel
  (session switch; an `on: no` for it written in `_olai/Settings.olai` is
  ignored with the reader-owner warning, and that is a scenario too) lists no
  outlines in the tree, the settings reader reports absent with every other
  row's applied patch standing; the `/trash` and `/agenda` pages and the Inbox
  and Pins sidebar entries say `the olai row is off` (they read the configured
  outline row off the claims cell and find no claim under it); any outline's
  address, the Inbox and Pins files included, is the "nothing by that name"
  page saying `no row claims `.olai``; turning it on restores
  them without reload; a file `notes.org` is never listed; `pdf` off removes
  `.pdf` files from the tree and `/media/` refuses them, and turning it on
  lists them again (ruling 4 applies to every kind row alike: a claimless file
  is not in the set); `outlines_create` with `olai` off refuses naming the
  row; two `Trash.*` files produce the finding.
  Step definitions `outline_list_steps.ts` and `viewer_steps.ts` referenced
  `files`'s per-kind test ids; point them at each row's `testids.ts`.
- `just typecheck-fast-remote`, `just test-fast-remote`, `just e2e-fast-remote`
  as you go; `just ci` on the pushed branch before asking for review.

## Docs (same PR)

- `docs/format.md`: "one suffix per outline row"; conventions by stem;
  `ambiguous-convention`; `format` setting meaning.
- `docs/architecture/overview.md`: the kinds table becomes "which row claims
  what"; the `format` row in the packages table loses "parse per file, the
  canonical writer".
- `docs/architecture/plugin-system.md` §12: `vault.file-kinds`, `files.kinds`,
  `navigation.pages`.
- `docs/running.md`: the `format` config paragraph.
- `docs/plugins/olai.md`, `hypertext.md`, `csv.md`, `image.md`, `pdf.md` new;
  `files.md`, `markdown.md`, `navigation.md`, `vault.md` updated. `just cordis-graph`
  reads the first paragraph of each, so write that paragraph as the row's purpose.
- `website/`: untouched.

## Order of work

Ruled: no transitional APIs, and no red step boundary. The order below is
additive until the switch, and the switch is one step, because dissolving a
closed union is atomic by nature. A step may be several commits; only the
step's last commit must be green under `just typecheck-fast-remote` and
`just test-fast-remote`.

1. **Contracts, additive.** `@olai/plugin-api`: `FileKinds`, `FileClaim`,
   `ComposedClaim`. `@olai/format`: `OutlineFormat`, `Claim` gains `noun`,
   `article`, `format?`, and the `Claims` type with its constructor. No reader
   changes; `FILE_KINDS` still decides everything. Green.
2. **The door, unread.** Vault: file-kind table in `openViews()`, `FileKinds`
   offered from `vault-setup`, `VaultSettings.claims` getter, the `file-kinds`
   cell with `outlineRow`, `vault.files.claims` and `kindOf` in the browser,
   `vault-revalidation` listening to `changes`. Nothing reads the table yet
   except its own tests. Green.
3. **Rows register, unread.** Six rows exist and register their claims on
   their server half: `olai` with `format` pointing at `@olai/format`'s
   `parseOutline`/`serializeOutline` (still there; a plugin importing the
   leaf is allowed), `markdown` narrowed to `.md`, and `hypertext`, `csv`,
   `image`, `pdf` with server halves only. `olai.yml` gains the `Files`
   section. The registry is now full and still unread. Green.
4. **The switch, one step.** Every reader of `FILE_KINDS` becomes a reader of
   `Claims`; `FILE_KINDS`, `FileKind` the union and its derivatives are
   deleted in the same step. Codec, ops writer and mints, conventions by stem
   and the `ambiguous-convention` finding, `address.ts` seam and the ops
   refusals, media route, `seal.ts` parameter, `wake.walks` and the doorbell
   `World`, git via `Ops`,
   chat via the vault's `outlineDiff`, and every browser caller via
   `vault.files.kindOf`. `parseOutline` and `serializeOutline` move into the
   olai row here (the row's `format` now points at its own module). Typecheck
   going red is this step's worklist; it is green at the step's end. Expect
   this to be the largest step by far.
5. **Faces move.** `files.kinds` and `navigation.pages` locations; the four
   body rows gain their browser halves; markdown's `faces.tsx` and files'
   `icons.tsx` and `kinds.ts` deleted. Green.
6. Tests, then docs, then `just ci` on the pushed branch.

Commit inside a step as often as useful; force nothing green mid-step. Steps
1 to 3 change no behaviour a test can see, which is what makes them safe to
land first.

## Cordis checklist

Each line is a guarantee, where it is enforced, and the test that shows it.
Reviewers read this list against the diff; a line without its test is not done.

| Guarantee | Enforced in | Evidence |
|---|---|---|
| A claim has one owner, stamped by the registry, never by the row | `openViews()`'s file-kind table, `kind` from `moduleOwner` | unit: a row passing `kind` is ignored; two rows claiming `.x` fails the second fiber, first keeps its files |
| A refused claim installs nothing and its cleanup deletes nothing of the winner's | one synchronous check-then-install over all exts | unit: image row claiming nine suffixes where one is taken leaves all nine unclaimed |
| A departed row's files leave the set | `register`'s finalizer, then `vault-revalidation` on `changes` | e2e: toggle `pdf`, `heads` loses the paths, `/media/` refuses; toggle on, they return |
| A probe in flight during withdrawal cannot pin the old table | codec reads `claims.current` at the top of each `match`/`decode` call, never caches | unit: withdraw between two probes, second probe drops the files |
| The reading published was validated with the table in force | `Reading` carries the `Claims` value it was validated with, not the getter | unit: reading's `claims` is a snapshot; toggling a row does not mutate a published reading |
| No format reads the registry it is registered into | `OutlineFormat.parse(file, contents, claims)` | fence: `packages/plugins/olai` imports no `FileKinds` outside `server.ts` |
| Git and chat never parse through the wrong format | git via `Ops.parserFor`; chat via the vault's `outlineDiff` procedure | fence: neither package imports `olai-plugin-olai` |
| Chat's diff survives the vault being absent | chat's browser holds the vault client through `Wired`; absent client draws "unreadable", never throws | browser test: vault client revoked, diff draws the absent sentence |
| Browser claims never go stale across a reconnect | `vault.files.claims` is a cell; reconnect resubscribes and takes a fresh snapshot; no consumer caches it | browser test: retire the wire, republish claims, tree redraws |
| A kind row's browser half degrades per component | `glyph` and `page` are separate components | e2e: files off, pdf page still opens at its address; navigation off, tree and rail withdraw as today and the pdf row's `glyph` component stays mounted without error (unit: contribution present, nothing reads it); files back on, glyph drawn without a reload |
| Withdrawing a face releases what it acquired | contributions to `files.kinds` and `navigation.pages` are scoped to the component activation | existing `Faces` withdrawal tests, extended to both locations |
| A mint whose row is off refuses, nothing written | `outlinePath` and `markdown_create` ask `mintExt` and refuse before staging | e2e: `outlines_create` with `olai` off |
| No default table a forgetting caller silently gets | `codecFor(kinds, claims)` has no default; `NO_CLAIMS` is a test-only export | typecheck |
| Two hosts in one process do not share a claim table | table minted inside `vault-setup`'s `apply`, like `openViews()` | existing two-host test in `views.test.ts`, extended |
| Only the claiming row spells its suffix | `packages/tests/kinds.test.ts` | the sweep |

## What not to do

- No module-level registry, no default `Claims` that a forgetting caller
  silently gets (`codecFor` argues this for `KindVocabulary`).
- No transitional compatibility API, however well named. A reader takes
  `Claims` or it reads `FILE_KINDS`; the two never coexist past step 4, and no
  shim bridges them in between. The order of work is arranged so none is
  needed.
- Do not keep a static suffix list "for the browser". The browser reads the cell.
- Do not add org, `.jsonl`, or a migration.
- Do not let a departed kind row leave its files stamped in the store cache as
  claimed; verify with a test that toggles the row and re-reads `heads`.
- Do not widen `files` back into a kind table under a new name.
