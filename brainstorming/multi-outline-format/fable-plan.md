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
| `olai` | new | `.olai`, `holds: nodes`, JSONL `parse` and `serialize` | nothing |
| `markdown` | existing | `.md`, `holds: text`, `kept` | document face, editor, glyph, noun |
| `hypertext` | new | `.html`, `text`, unkept, fetched | sealed-frame face, glyph, noun |
| `csv` | new | `.csv`, `text`, unkept | table face, glyph, noun |
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
5. **`olai` can be switched off.** Off: every `.olai` is unclaimed, mints refuse
   naming the row, the trash, inbox, pins and agenda pages say the row is off.
   The plugins panel gets a `switchHint` saying so. Honest beats safe here.
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
  readonly kind: string                       // "outline", "document", "image" …
  readonly exts: readonly [string, ...string[]] // matched exactly, first is what a mint writes
  readonly holds: "nodes" | "text" | "bytes"
  readonly kept: boolean
  readonly fetched: boolean
  readonly noun: string                       // "outline", "page"
  readonly article: "a" | "an"
  readonly format?: OutlineFormat             // required iff holds === "nodes"
}
export interface FileKinds {
  readonly current: () => ReadonlyMap<string, FileClaim>   // by kind
  readonly changes: Stream.Stream<void>
  /** Claim, for as long as the calling plugin is loaded. A suffix or kind
   *  already claimed FAILS the calling fiber (the `Kinds.register` rule). */
  readonly register: (claim: FileClaim) => Effect.Effect<void, never, Scope.Scope>
}
export const FileKinds = serviceTag<FileKinds>("vault.file-kinds")
```

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
- `@olai/ops` `codec.ts`: `match` asks `fileKind(claims.current, path) !== null`.
  `decode` is `parserFor(claims, path)?.parse(path, contents, claims)`, the
  one `claims` value read at the top of the call. `byName` through `unkept`. All via
  the getter, read per call.
- `@olai/ops` writer (`ops.ts:668`, `following.ts:138`): serialize through
  `parserFor(claims, planned.file).serialize(nodes)` and read back through
  `.parse(planned.file, text, claims)`, the same `claims` value the planner was
  judged against. A planned file whose suffix has no format is a defect
  refusal, same shape as the existing read-back guard.
- `@olai/ops` `plan.ts` `outlinePath`: takes `mintExt(claims, "outline")`
  from the row named by `format`; when that row is not claiming, refuse with
  `the olai row is off, so no outline can be created`. Same for
  `markdown_create` via `mintExt(claims, "document")`.
- `@olai/format` `node.ts` conventions: `TRASH`, `INBOX`, `PINS`, `PROPERTIES`
  become stems; `outlineCalled(files, stem)` matches by `stemOf` over
  `holds: nodes` files. `TRASH_FILE` and `isTrashed` take claims. The
  `ambiguous-convention` finding lives in `rules.ts`.
- `git` (`packages/plugins/git/src/ledger/committed.ts:274`): parse the
  committed side through the format for that path, reached through `Ops`
  (`Ops` gains `parserFor`). `pending.ts` and `pending.testlib.ts` follow.
- `chat` (`packages/plugins/chat/src/browser/chat/outline.ts`): the browser
  must not parse. Add a procedure `outlineDiff({ path, oldText, newText })` on
  the VAULT's file surface (`packages/plugins/vault/src/file-surface.ts`),
  answered with `parserFor(claims, path)?.parse(path, text, claims)` on each
  side and `@olai/format`'s `changesOf`, `claims` being the vault's current table.
  Chat's browser calls it through the `Wired` client it already holds and draws
  the answer. Nothing in `chat/src/server.ts` or `chat/src/wire.ts` changes:
  another branch (`chat-sidebar-ux`) is rewriting those files, and parsing is
  the vault's business anyway.
- `kolu/src/wake.ts` and `odu/src/wake.ts`: `wake.kinds` was typed `NodeKind`.
  It becomes `ReadonlyArray<string>` checked at wake time against
  `claims.current` (`holds === "nodes"`), refused in words otherwise. Keep the
  screenshot defect (2026-09-01) in the test.
- `packages/plugins/vault/src/http/media.ts`: `isAsset` asks `isFetched(claims, path)`.
- `packages/surface/src/seal.ts` interpolates `FILE_EXTS` into the sealed
  frame's script. It becomes a parameter; the caller (`markdown`'s hypertext
  face, after the move: the `hypertext` row) passes the current suffix list.
- `packages/surface/src/plugins.ts` and `packages/surface/src/attach.ts`:
  `attach.ts` is a DIFFERENT list (what a person may hand an agent) and stays;
  check what `plugins.ts` reads and route it through claims.

## Browser plumbing

- `vault.files` (`packages/plugins/vault/src/browser/state.ts`, contract
  `contract.ts`) gains `claims: Accessor<Claims>` fed by a new `file-kinds`
  cell on the vault surface (the table minus `format`), and `kindOf(path)`.
  Every browser caller of `fileKind` goes through it: `navigation/src/routes.ts`,
  `files/src/fileTree.ts`, `files/src/Files.tsx`, `files/src/contracts/completing.ts`,
  `files/src/file/making.ts`, `markdown/src/browser/document-route.ts`,
  `markdown/src/browser.tsx`, `outlines/src/browser.tsx`, `search/src/browser/KindSelector.tsx`,
  `chat/src/browser/chat/ToolFrame.tsx`, `outlines/src/browser/Nothing.tsx`, `trash/src/Entry.tsx`.
- `files` owns location `files.kinds`, contribution
  `{ kind, glyph: () => JSX.Element, noun, article, testid }`, keyed by kind.
  `Files.tsx`, `Rail.tsx` and `fileTree.ts` read it. Every path in `heads` is
  claimed by some row's SERVER half, so the only way a listed path has no glyph
  contribution is that row's BROWSER half being absent (failed to load, or the
  tab has not fetched it yet). For that case draw the plain-file glyph and the
  noun from `claims.byKind.get(kind)?.noun`. A row that is OFF has no files in
  `heads` at all (ruling 4), so the tree never shows them and needs no "row is
  off" drawing. `contracts/icons.tsx` and `contracts/kinds.ts` are deleted;
  each glyph moves to its row.
- `navigation` owns location `navigation.pages`, contribution
  `{ kind, page: (address) => JSX.Element, edits: boolean }`, keyed by kind.
  `routes.ts` picks the page by `kindOf(path)`. A path the directory does not
  hold (its row is off, or nobody ever claimed the suffix) is the existing
  "the directory holds nothing by that name" page, which now carries the kind
  string it was asked for. A path that IS held but whose kind has no page
  contribution mounted (server half claims, browser half absent) draws a page
  saying which row's browser half is missing. Those are the only two cases.
  `markdown/src/browser/document/faces.tsx` is deleted; `DocumentPage.tsx`
  becomes markdown's own contribution.
- New rows `hypertext`, `csv`, `image`, `pdf` under `packages/plugins/<row>/`:
  `package.json` (`olai-plugin-<row>`, exports `./server`, `./browser`,
  `./testids`), `src/server.ts` (`needs: [FileKinds]`, one `register`),
  `src/browser.tsx` (`needs: [Offers, fileState-ish]`, contributes to
  `files.kinds` and `navigation.pages`), `src/testids.ts`. Move the faces out of
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
  `{ kind: "outline", exts: [".olai"], holds: "nodes", kept: true, fetched: false, noun: "outline", article: "an", format }`.
  No browser half. `@olai/format`'s tests that exercised `parseOutline` and
  `serializeOutline` move with them or import the row's pure module (a static
  import of pure functions is allowed by `cordis.md`).
- `packages/bundle/olai.yml`: new section `Files` holding `olai`, `markdown`,
  `hypertext`, `csv`, `image`, `pdf`, in that order. `olai` carries
  `profiles: [surface, test-minimal]` like vault and a `switchHint`. Run
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
- e2e (`packages/tests/features/`): a vault served with `olai` off lists no
  outlines in the tree, `/trash` and the inbox say the row is off, and an
  outline's address is the "nothing by that name" page; turning it on restores
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

1. `@olai/format`: `Claims`, `OutlineFormat`, readers take claims. Typecheck
   goes red everywhere; that is the worklist.
2. `@olai/plugin-api`: `FileKinds` contract.
3. Vault: door, settings getter, codec, revalidation, media route.
4. `olai` row; move parse and write; ops dispatch; conventions by stem; mints.
5. `git` and `chat` rerouted.
6. Browser: `vault.files.claims` cell, `files.kinds`, `navigation.pages`.
7. `markdown` narrowed; four body rows extracted.
8. Tests, then docs, then `just ci`.

Commit after each step. Each step should leave `just typecheck-fast-remote` green.

## What not to do

- No module-level registry, no default `Claims` that a forgetting caller
  silently gets (`codecFor` argues this for `KindVocabulary`).
- Do not keep a static suffix list "for the browser". The browser reads the cell.
- Do not add org, `.jsonl`, or a migration.
- Do not let a departed kind row leave its files stamped in the store cache as
  claimed; verify with a test that toggles the row and re-reads `heads`.
- Do not widen `files` back into a kind table under a new name.
