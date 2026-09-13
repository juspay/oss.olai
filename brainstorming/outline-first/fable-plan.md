# Outline first, files second

An implementation brief. One PR, three parts, in this order: the dead-link reading, the removal of the `doc` field, the outline-first sidebar. Line numbers are as of `076635710`; treat them as pointers, not gospel.

Prototype of the sidebar, clickable: <https://claude.ai/code/artifact/d967e1d1-a6c1-445a-9fdf-04858ec37273>. Flip the switch at the top to compare with today's column.

## The ruling (the human, 2026-09-13)

Olai serves outlines and files, and today draws both as peers in the sidebar. That is two front doors, and it confuses. The philosophy adopted is **the outline is the map, the files are the territory**: an outline indexes, dates, marks and links; a `.md`, a `.pdf`, a picture holds material the outline points at. Concretely:

1. **The sidebar's tree lists outlines and nothing else.** Every other served file lives in a collapsed **Reference** section under it, grouped by folder. There is no preference to bring the old tree back.
2. **The `doc` field is removed from the format.** Nothing special attaches a document to a node any more; a link in the note is the attachment. No compatibility shim: a record carrying a top-level `doc` is a `bad-record` like any other unknown key, exactly as `titel` is. A vault that wants to keep the value moves it under `custom` (one `jq` line, below), where it is an ordinary property that the undeclared-key guess draws as a door.
3. **A relative link that lands on nothing served is said out loud**, everywhere the link is read: under the row, on the node page, on `outlines_read`'s answer, and as a nudge on the answer to the write that introduced it, with a did-you-mean. It is never a refusal and never breaks the file.
4. Not in scope: any migration of legacy `.md` notes into nodes; the `doc` **property kind** (`{"type":"doc"}` in `_olai/Properties.olai`), which is a declared fence and stays; the daily-note convention, which the day page already reaches.

Preserve Cordis adherence throughout (see `CLAUDE.md`): live values cross package boundaries through declared services and locations, every contribution has an owner and a lifetime, withdrawal order is kept.

---

## Part C first: the dead-link reading

Done first because Part B leans on it: once `doc` is gone, a link in a note is the only way a node attaches a file, and the delete guard and the sidebar both read links.

### What it is, and what it is not

**Not a validator finding.** The validator has no warning tier and the removal of one is documented (`packages/format/src/verdict.ts:37-56`, "every finding is per file"). Any `OutlineError` is blamed on its own site (`verdict.ts:376`), which withholds the file's page (`set.ts:336-355`) and refuses its writes (`verdict.ts:176-193`). A dead link must not do any of that. A link to a file you are about to create is legitimate.

**A derived reading**, like blockedness and backlinks: computed from the forward links the set already keeps and the paths it serves, stored nowhere. The forward reading exists: `recordLinks` at `packages/format/src/documents.ts:645` (title and note through `linksIn`, resolved beside the defining outline) and a document body's `links` at `document.ts:390`. Existence is deliberately not asked there (`documents.ts:206-211`), and `markdown-ui`'s `resolveDocument` says the same (`rewrite.ts:162-165`). This part adds the one reader that asks.

### The rule

A link written in a node's title or note, or in a document's body, is **dead** when it is relative (not `http(s):`, not `/`, not a bare `#fragment`, per `refusedHref` at `documents.ts:163`), resolves beside the file it was written in (`resolveRelative`, `documents.ts:79`), and the resolved path is not a file the directory serves at this revision. A link to a served file whose heading fragment does not exist is not this rule (headings are already the document's own business).

The did-you-mean: among served paths, those whose basename (`paths.ts:74`) equals the link's basename, spelled **relative to the writing file** (the inverse of `resolveRelative`; `retargetRelative` at `documents.ts:89` is the nearest existing arithmetic and is otherwise being deleted in Part B, so lift what it knows before it goes). One candidate is the suggestion; several are listed; none falls back to `didYouMean(resolved, servedPaths)` by edit distance (`suggest.ts:102`, the pattern `typing.ts:1432-1446` already uses for a `doc`-kind value).

### Where it is read

- **A new format reading**, in `@olai/format`, beside `backlinks.ts`: given a located record (or a document face) and the served path set, answer its dead links: `{ written, resolved, suggest: string[] }[]`. Pure; unit-tested in `packages/format/src` with the `..`-clamp case, a percent-encoded name, an angle-bracket name, a link with a fragment, and the basename did-you-mean across folders.
- **`outlines_read`** (`packages/plugins/outlines/src/tools.ts:111-115`) carries it on the answer, omitted when empty, so an agent that reads the node back sees the finding beside the note it wrote. Same for `outlines_subtree`. `markdown_read` carries a document's.
- **The node page and the row.** Draw one quiet line per dead link under the note, in the alarm tone but toned as an aside, not a refusal: *link resolves to nothing served* + the resolved path in mono + *did you mean `../notes/x.md`?*. The `outline.row.aside` slot (`browser/Asides.tsx`) is the right seat if it fits one line per finding; otherwise a sibling of the `see` line inside `NodeBody.tsx`. The node page's `Shown` arm (`page.ts:190-196`) carries only backlinks today; add the reading to what the page model answers rather than recomputing it in the browser.
- **Inline in the rendered note.** `markdown-ui` already rewrites an unresolvable picture to a visible span (`rewrite.ts:112-142`, `undrawnPicture`). Give `resolveDocument` an optional `serves(path)` predicate from the renderer's props and, when it answers no, keep the anchor but mark it (`data-dead`, a wavy alarm underline, a title with the resolved path). The browser has the served path set through `vault.files` (`Directory.paths`, `packages/plugins/vault/src/browser/directory.ts:169-224`); thread it where `from` is already threaded (`Markdown.tsx:53`). A renderer with no predicate draws as today.
- **The nudge.** `Plan.nudge` (`packages/ops/src/plan.ts:152-154`) reaches every write answer for free (`ops.ts:649, 841-846`). `planTitle` (`plan.ts:192`), `planDesc` (`plan.ts:221`), `outlines_add`, `outlines_update`, `capture_add`, `planWriteDocument` (`plan.ts:5445`) and `planCreateDocument` (`plan.ts:5507`) set it when the text they landed holds a dead link that the text they replaced did not. One sentence per link, joined the way batch nudges already are (`plan.ts:4646-4648`). The browser draws it through the existing aside tone (`SaidLine.tsx`, `RowEditor.tsx:301-304`); the document editor's save gets the same.
- **The verbs say the rule.** One sentence on `outlines_desc` (`outlines/src/tools.ts:152-157`), `outlines_title`, `outlines_add`, `outlines_update`, `markdown_write`, `markdown_create`, and on the `desc` / `title` field annotations in `packages/format/src/writing.ts:90-92, 396-421, 1129-1182`: *A relative link resolves beside the file this node lives in (or this document), not beside the reader; the answer names any link that lands on nothing served.*

### Tests

- Unit: the reading (above); `plan.test.ts` cases for the nudge on desc, title, add, document write; a case proving a link to a file created later stops being dead on the next revision without a write.
- e2e (`packages/tests/features`): a new feature. An agent writes a note with `[x](nix-flakes.md)` from `projects/olai.olai` while `notes/nix-flakes.md` is served; the row draws the finding with the suggestion; `outlines_read` answers it; the write answer carried the nudge (`the nudge says …`, precedent `keyboard_editing.feature:529-562`); creating `projects/nix-flakes.md` clears it live; the same for a document body through Save. Cover the `..` climb-out and a link with `%20`.

---

## Part B: remove the `doc` field

### On disk

The field leaves the record schema and the canonical write order in one commit; `write.test.ts:217` and `parse.test.ts:212` hold schema and order to each other. A file carrying `doc` at the top level is refused as `bad-record` naming the key, like any unknown key. For a vault that wants to keep the values:

```sh
git ls-files '*.olai' | xargs -I{} sh -c \
  "jq -c 'if .doc then .custom = ((.custom // {}) + {doc: .doc}) | del(.doc) else . end' '{}' > '{}.tmp' && mv '{}.tmp' '{}'"
```

(Under `custom`, `doc` is an ordinary key; the undeclared-key guess resolves it beside the writing file and draws a door when the file is served, `docs/format.md:170`.) Put this recipe in `docs/format.md` beside the `.jsonl` rename recipe.

### Checklist

`@olai/format` (`packages/format/src`):
- `node.ts:266` the field; `node.ts:341` its `DOORS` entry (the `satisfies` forces both).
- `documents.ts:66-73` `docOf`; `:89-111` `retargetRelative` (lift the relative-spelling arithmetic into Part C first); `:645-648` the `attached` arm of `recordLinks` (a mirror now answers `NO_LINKS` directly); `:660-663` `pathAddress`. Barrel exports at `index.ts:270, 281`.
- `rules.ts:355-385` `reportDocs` and the `missing-doc` code at `errors.ts:135-136`. **Keep** `markdownPaths` (`rules.ts:106`): the `doc` kind reads it (`rules.ts:471-472`).
- `validate.ts:388` and `incremental.ts:308`: the calls. **Keep** the `known` / `lost` / `carriedDocuments` carry (`incremental.ts:287-303`) since `walkingProps` (`:323`) folds it in for the kind.
- `changes.ts:320` (`changed.has("doc")` in `sortOf`), `filter.ts:328` (`has:doc`), `searching.ts:250` (the agent-facing teaching string listing `has:doc`), `reading.ts:503` (`NOT_PROJECTABLE.doc`, type-forced).
- Prose to reword: `documents.ts` header and `:119, 194, 375, 390, 397, 465, 626`; `backlinks.ts:214, 222`; `pointing.ts:5, 13`; `document.ts:128, 256, 433`; `set.ts:516, 532`; `page.ts:482`; `meaning.ts:38, 171, 275-284, 420-447, 491` (kind arms, code stays); `typing.ts:382`; `incremental.ts:52-55, 91, 170, 301-307, 368, 433`; `filter.ts:2276`; `validate.bench.ts:44-51, 121-123`; `pointing.bench.ts:10, 42`; `README.md:92, 151`.
- Tests: `documents.test.ts:42-71, 269`; `validate.test.ts:117, 557-590`; `errors.test.ts:129, 189, 263`; `verdict.test.ts:110`; `incremental.test.ts:38, 456-510` (keep `:630-635`, kind); `incremental.testlib.ts:567, 639, 643, 692`; `pointing.test.ts:298, 339, 344` and `pointing.testlib.ts:30, 242, 266, 420-421`; `filter.test.ts:41, 149`; `document.test.ts:30, 303`; `set.test.ts:120`; `corpora.testlib.ts:20`.

`@olai/ops` (`packages/ops/src`):
- `plan.ts:3816-3823` `carryingDoc` and its call sites `:3133` (move), `:3753` (archive), `:4073` (untrash); the `retargetRelative` import at `:76`.
- `plan.ts:3536` the merge nudge clause "its document".
- `plan.ts:5584` the field arm of `namingDocument` (the `files_delete` guard) and the `docOf` import at `:46`. **Keep** `:5579-5581, 5585-5600`, the kind arm. **Add** the link arm: a document is still named while any live record's note or title links to it, or any served document's body does, read off `pointing` (`pointing.ts:138`, `referrersTo` at `backlinks.ts:259`). The refusal at `:5720` names the record and says `link` where it said `doc`. Trashed records do not hold a file (`backlinks.ts:292` already excludes them).
- `plan.ts:1640-1648` the repeat rule's "does not carry the document" paragraph (prose; the code never copied it).
- Prose: `plan.ts:317-318, 1051-1053, 2389-2390, 2880, 3098-3103, 4318-4321, 4461`; `codec.ts:68`; `query.ts:1249`.
- Tests: `plan.test.ts:2296, 3044-3051, 3144-3150, 3694-3722, 3818-3827, 5017-5070` (keep `:5086-5094`, kind); `codec.test.ts:206, 212`; `ops.test.ts:154`. New: the link arm of the delete guard, from a note and from a document body.

Writer: `packages/plugins/outline-olai/src/format.ts:349` (`"doc"` in `ORDER`).

Server: `packages/server/src/file-kind-formats.test.ts:50, 63`.

Outlines plugin (`packages/plugins/outlines/src`):
- `index.ts:50` the `outlines.document-reference` location; its only reader is `NodeBody.tsx` and its only contributor is markdown's `DocRef`, so the location goes too. `browser.tsx:43, 174`.
- `NodeBody.tsx:75, 79, 277-281, 316-325` and header prose `:36, 96`. The "a node with a `doc` and no note has no pilcrow" argument in `browser/body.ts:8` dies with it.
- `tools.ts:211` (`outlines_duplicate` description). Prose: `Tree.tsx:230`, `Note.tsx:21`, `doors.ts:15`, `props/PropsDrawer.tsx:249-250`, `contracts/property-values.ts:30`. Test fixture `props/drawer.test.ts:49`.
- `docs.md:7, 18`.

Markdown plugin (`packages/plugins/markdown/src`):
- `browser/document/DocRef.tsx` whole file; `browser.tsx:39, 41, 108` (keep `:109`, the kind's `propertyRoutes`); `testids.ts:7-8`; `tools.ts:39`.
- Prose: `browser/document/documents.tsx:37, 45, 104-107` (the shared per-file subscription stays; its justification changes to "several rows can link one document"), `ready.ts:4`, `BodyRefused.tsx:6`, `Referrers.tsx:5, 19-23`, `surface.ts:40`, `wire.ts:40`. `docs.md:17`.

Leave alone: the edit verbs `doc` / `docNew` / `docDay` (`packages/surface/src/edit.ts:888-904`) and the ops `doc` / `create-doc` (`writing.ts:871, 900`) are whole-file document writes and unrelated. Chat's `attachments.ts`, `prompt.ts`, `teaching.ts` do not mention the field.

Passing mentions to reword: `packages/appearance/src/scale.ts:283`; `packages/markdown-ui/src/pipeline.ts:36`, `rewrite.ts:59`, `slugs.test.ts:97`; `packages/format/src/frontmatter.ts:165`; `packages/surface/src/seal.ts:277`, `edit.ts:128`; `packages/plugins/chat/src/browser/chat/attention/notice.ts:56`; `packages/plugins/trash/src/tools.ts:31`; `packages/plugins/files/src/file/delete.ts:25`; `packages/plugins/files/src/tools.ts:36`.

### e2e

- Fixture `packages/tests/fixtures/good/house.olai:4` drops `"doc":"finishes.md"`; give `install` a note linking `[finishes](finishes.md)` instead so the scenarios that hang off it can be rewritten rather than dropped.
- `documents.feature:253, 290, 304, 348` become link scenarios: the note draws the link, following it opens the document, an unreadable target says so inline (Part C's dead-link line covers "not served"; "served but unreadable" keeps the existing `BodyRefused` behaviour on the document's own page).
- `document_editing.feature:303-311` and `file_delete_concurrency.feature:13, 17` assert the new refusal wording (`link`, not `doc`).
- Dead steps in `document_steps.ts:342-411` and `world.ts:376-379, 2135-2137`; the trash step at `live_steps.ts:242-243` that re-targets `moved["doc"]`.
- Feature prose: `duplicate_subtree.feature:25`, `move_to_picker.feature:172, 277`, `split_and_merge.feature:65`, `a_frame_leaves_it_standing.feature:113-114`, `html_previews.feature:1939`, `fixtures/good/kitchen-sink.md:123`.

### Docs

- `docs/format.md:41` (the Fields row), `:153`, `:371`, `:383`, `:397` (keep the property clause), `:400`, `:477, 480`, `:508`; add the `jq` recipe. `:119, 140-167, 187, 193` are the kind and stay.
- `docs/search.md:36` (`has:doc`). `docs/editing.md:534`. `docs/architecture/overview.md:102`. `docs/architecture/e2e-coverage.md:163`.
- `website/index.html:208` (`.docref` rule) and `:418` (the `cabinets.md` line in the hero mock): a picture of something a person sees is now wrong, so it goes. Replace with nothing, or with a note line under the pilcrow that holds a link.

---

## Part A: the outline-first sidebar

### What the column draws, top to bottom

Unchanged: the top entries (Inbox, Agenda), the calendar, Needs you, the pinned shelf. Then, replacing today's single tree:

1. **Outlines.** The `files` region draws a tree of served files whose claim `holds === "nodes"` (`Files.tsx:452`), and the folders on the way to them. A folder with no outline anywhere beneath it is not drawn. Labels are stems already (`fileTree.ts:96`, `stemOf`); nothing to change there. Sort files by stem now that `a.md` and `a.olai` no longer sit side by side (`fileTree.ts:100-112`). Folders still start collapsed and the open file's ancestry still wins (`Files.tsx:68-73, 384-391`); `openFolders` keys stay directory paths, so a folder open in both trees is one preference.
2. **Reference.** A second region under the outlines, with a header row that is itself the fold: a triangle, the word *Reference*, and a count of files at the right in mono. Collapsed by default; the open state is a browser preference beside `openFolders` (`fold/folders.ts`, same circuit, its own key). Inside: every served file whose claim does not hold nodes, `_olai/` excluded, grouped by folder with the same `Dir` / `File` rows and the same glyphs (`glyphs.tsx`, `drawingOf`). Nested folders here fold with the same `openFolders` set. A file open in a pane lights its row (`aria-current`), and opening one expands Reference and its ancestry the way a folder chain expands today. The count is the number of files, not folders.
3. **The `_olai` group** exactly as it is (`Files.tsx:187-217`).
4. **The new-file doors** (`files.types`: `+ New outline`, `+ New document`) stay under the outlines tree where they are (`Files.tsx:168-169`). `+ New document` mints into Reference; the sidebar lists it on the same frame and Reference opens to show it.

The rail (`Rail.tsx`) keeps its two buttons.

Empty states: a directory with no outline draws the *Outlines* region empty with only the doors; a directory with nothing but outlines draws no Reference header at all rather than *Reference 0*.

### Where the walk lives

Split `fileTree(claims, files)` (`fileTree.ts:131`) by claim: one walk over files whose kind holds nodes, one over the rest, each dropping empty folders (which `put`/`freeze` do not do today for the "folder with only unclaimed files" case, so add it). `dirsIn` prunes fold memory against the union. Unit tests in `fileTree.test.ts` for both walks, the empty-folder rule, and the count.

Everything stays inside the `files` plugin: the region contribution at `browser.tsx:60` grows a second section in the same `Body`, owned by the same activation, withdrawn together. No new service and no new slot: the Reference section is a drawing of `vault.files`, and `pins`, `capture`, `trash` keep their own regions untouched.

### e2e

Steps that count or address the tree assume one list. Update, do not fork:
- `outline_list_steps.ts:50` (`has {int} entries`) counts outlines only. Add *the reference section lists {int} files*, *the reference section is collapsed / expanded*, *I expand the reference section*, and re-point `:318, 329` (document link shown / hidden) at the Reference section.
- `viewer_steps.ts:49` (`the {string} rows listed are {string}`) scopes non-outline kinds to Reference.
- `file_kind_steps.ts:56-63` (unclaimed absent) scopes to both regions; `:43` still asserts the whole `sidebarFiles` region absent.
- `feed_steps.ts:134` (`the vault group sits below the file tree`): below Reference now.
- Features to walk through: `serve_a_directory.feature:9-57` (the `Daily` folder holds `.md` only, so it moves to Reference; `has 2 entries` changes meaning), `documents.feature:19-26, 142, 385-393`, `document_editing.feature:123-148, 335-341`, `pdf_csv_and_pictures.feature:39-54, 59, 107, 135`, `file_kinds.feature`, `folds_are_remembered.feature:103-110` (add: Reference open survives a reload), `the_chrome_holds_still.feature:26-49`, `the_sidebar_sticks.feature:22-51`, `new_outline.feature`, `new_file_lifecycle.feature` (a new document lands in Reference and Reference opens), `html_previews.feature` and the others that `expand the folder "notes"` as setup (they now expand Reference first).
- New: a vault with no outline; a vault with nothing but outlines; a folder holding both kinds appears in both regions; opening a document from a link on a node page lights its row inside Reference.
- `packages/tests/README.md:418-449` selector notes; add the Reference testids.

### Docs

- `docs/editing.md:396-406` (what the sidebar leaves out) gains the section on the two regions and the philosophy in two sentences; `:410` (the shelf sits "between the calendar and the file tree"); `:498-504` (the doors); `:524` (a new document is listed in Reference); `:530`.
- `packages/plugins/files/docs.md` (`docs/plugins/files.md`) `:3-6, 16-18`: the two walks, the Reference preference. `packages/plugins/sidebar/docs.md` `:17-19` if the region contract changes (it should not).
- `docs/index.md:10, 65, 69` glosses only if their wording goes stale (the docs test holds glosses above 40 characters).
- `docs/architecture/e2e-coverage.md`: the new scenarios.
- `website/index.html:746` (`files` → "The directory tree.") becomes "Outlines in front, everything else under Reference." That and the hero mock (Part B) are the only website touches: the reason to try olai did not change, one picture did.

---

## Delivery rule

This is **one PR, opened as a draft, and never merged by the implementor.** The implementor opens the draft, keeps it green, and hands it over; marking it ready for review and merging are the human's calls. Do not squash, rebase onto master after review has started, or merge on a green `just ci`.

## Order of work and checks

Commits in this order so each lands green: (1) the dead-link reading and its surfaces, (2) the `doc` field removal with the delete guard's link arm, (3) the sidebar. Run `just typecheck-fast-remote`, `just test-fast-remote`, `just e2e-fast-remote` per commit against the working tree; `just ci` on the pushed branch before asking for review. Audit `docs/architecture/e2e-coverage.md` for the workflows above and add what is missing rather than trusting the existing suite.
