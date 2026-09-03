# Editing UX vs Workflowy

**TL;DR.** Olai's outline is a viewer you switch into an editor; Workflowy is an editor you can view. A 2026-09-02 browser drive found seventeen concrete differences. They are being closed in three phases. Phase 1 (make the `<input>` honest) is three PRs shipped and one PR left: the paste parser and the format keys. Phase 2 (the caret walks the tree) and Phase 3 (agenda and day pages become editable) are not started and wait on the human's word. Both were recut on 2026-09-03 after a read of the client code showed the original plan's Phase 2 asking for a state the editor does not have, and its Phase 3 leaning on a door that does not exist; the recut is smaller and says what is structural.

Driven 2026-09-02. State as of 2026-09-03.

This is not a rewrite of [editing-web.md](editing-web.md), which is the 2026-08 research the editor was built from. This is what a Workflowy hand hits in the browser after that editor shipped, and the plan for closing the gap.

## Method

`just serve` on the `good` fixture, Playwright driving `http://127.0.0.1:7788` the way a person types: click, type, Enter, Tab, paste, arrows, chords, agenda, day page, phone viewport.

The 2026-08-12 inventory in editing-web.md listed missing *features* (paste-as-tree, format keys, delete key, bullet types). The finding here is that the missing menu entries are not the problem. The problem is the mode.

## Diagnosis

Three deliberate design choices compose into the feel:

1. **A title is an `<input>`**, mounted on click, unmounted on blur.
2. **Nothing is optimistic.** Structural keys are intents; the row moves when the file says so. Empty titles are illegal on disk, so a new row is a local draft until it has words.
3. **Only outline and zoom pages are editable.** Agenda and day pages are query views. Notes are a second field behind a ¶.

Workflowy's model, driven from its help pages: click any character and the caret is there; Enter always makes a bullet, even an empty one; paste of indented text is a tree; notes sit under the bullet; zoom, collapse and search are keys; every filtered view is still the same document.

Olai copied Workflowy's *chords* and not its *continuity*.

## The root constraint

Three rules hold the current design up:

- The format will not hold an empty title.
- The wire will not echo a write.
- The title editor will not be `contenteditable`, because a title is one verbatim string and a live frame would fight the caret.

All three are right for git and for agents. The plan keeps the first two (empty-on-disk stays illegal; the file stays the truth) and, in Phase 2, replaces the third with something that is still not `contenteditable`.

## What is already fine

Do not "fix" these. They match Workflowy, or they are olai on purpose:

- Tab / Shift+Tab indent, ~110ms to land.
- Ctrl/⌘+Enter completes. Glyph click zooms (Workflowy too).
- `((` mirrors, `!` dates, `#`/`@` tags, ⌘⇧D duplicate, ⌘⇧M move-to, drag-a-bullet, the five multi-select gestures, the ••• menu.
- Backspace at column 0 merges.
- Split at the caret keeps children on the first half.
- No last-write-wins, no `#copy` tag, no completing-a-parent-completes-children. Keep those refusals.

## Problems found, and where each stands

Severity: **blocker** = a Workflowy hand cannot do the thing at all. **pain** = they can, but the feel is wrong every time. **friction** = an extra click or a missing chord. **gap** = listed in 2026-08-12 and still missing.

Each problem ends with its state. "Fixed" names the PR. "Phase 2" or "Phase 3" means the fix is in a later phase below. "Stays" means it was ruled to remain as it is.

### A. There is a mode

**A1. Typing does nothing until you click a title.** Blocker as found.
Focus is `BODY`; there is no caret in the outline. Workflowy's caret is always in some bullet.
*State: stays.* The proposed fix (a bare keystroke opens the last-focused or first visible row) was rejected by the human on 2026-09-03: "I reject A2, it is silly." Guessing which row a keystroke meant is the silliness. A caret gets somewhere by clicking, or by arrowing in Phase 2.

**A2. A click does not place the caret.** Blocker.
Clicking 8px into an 18-character title put the caret at 18. The click's X was thrown away when the span was torn down and the `<input>` mounted at `value.length`.
*State: fixed, [#475](https://github.com/juspay/olai/pull/475).* A press on a title measures the source string in that title's font and opens the input at that offset. A press on the filler to the right of the title is still the end of the line. The input now uses the title's own type and leading, so the row does not jump on open.

**A3. The read face and the edit face are different documents.** Pain.
`#home` is a coloured chip until click, then raw `#home` in an input. A note's `**walnut**` renders bold one click earlier and shows as source in the textarea.
*State: stays, by design.* docs/editing.md says it plainly: what you type is the source, and the rendering comes back the moment you leave. Phase 2 keeps this rule.

**A4. Left/Right at the ends of a line do not leave the row.** Pain.
Workflowy treats the outline as one text stream: Left at column 0 is the end of the bullet above. Olai's row-to-row arrows are Up/Down only, and each hop closes one input and opens another at the end.
*State: Phase 2.*

**A5. Escape throws the draft away.** Friction.
Typed ` xyz` on `garden`, pressed Escape: the title is `garden #outdoors` again. Workflowy's Escape is search; letters already belong to the document.
*State: open, not in any phase.* Escape stays "drop what you were typing" per docs/editing.md. It is listed in the chord table below.

### B. Blank bullets cannot exist

**B1. Enter Enter Enter does not make a list.** Blocker.
Five Enters made one ghost. You could not lay out a skeleton and fill it in, which is what outlining is.
*State: fixed, [#477](https://github.com/juspay/olai/pull/477).* Enter on an empty draft parks it and opens the next one. Leaving an empty draft (blur, click-away) parks it too; Escape drops the one you are in; closing the page drops them all. Disk still sees nothing until a draft has a title.

**B2. Enter at column 0 does not insert above.** Pain.
Enter at the start of a titled row put a ghost after the entire subtree, at the bottom of the page.
*State: fixed, [#477](https://github.com/juspay/olai/pull/477).* Enter at column 0 opens a draft above the row; the words stay put. The wire gained a `before` anchor, judged where every other placement is judged. Enter at end of line still adds after the subtree.

**B3. Splitting a parent teleports the caret.** Pain.
Split a section title in the middle: the first half stays the section with its children, the second half appears as a new row below the whole subtree, a screen away. The semantics match Workflowy; the teleport is "no optimistic UI" plus "sibling after subtree".
*State: Phase 2, PR 8.* A placement question before any UI: see open question 6.

**B4. Clearing a title is refused.** Pain.
Select-all, Backspace, Enter: "a node needs a title". Workflowy's blank bullets are first-class.
*State: stays.* Empty titles on disk are the root constraint. Blanks are local drafts only (B1, and Phase 2's local slots).

**B5. The new-row ghost is not a bullet.** Pain.
Hollow circle, muted placeholder, no Tab until it has a title.
*State: partly fixed by #477* (ghosts persist and can be clicked back into). A draft that takes Tab, indent and arrows like a real row is *Phase 2*.

### C. An `<input>` cannot be a tree

**C1. Paste of a nested list is one line.** Blocker.
A four-line tab-indented clipboard landed as one flattened ghost. Newlines cannot live in an `<input>`. This is the capture path Workflowy is loved for.
*State: open, Phase 1 PR 3.*

**C2. Ctrl+B / Ctrl+I do nothing.** Gap.
Titles already render `**bold**`. The keys should write the markdown the renderer already reads.
*State: open, Phase 1 PR 3.*

**C3. Long titles ellipsize on the read face.** Reading complaint, not an editing defect.
A row is one line under the quiet-outline ruling, so a long title reads as `order the...`, harder on a 390px phone. The edit face is unaffected: the editor's `<input>` carries no truncation and a native single-line input scrolls under the caret, so the text at the caret is always visible.
*State: stays; see open question 3.* An earlier claim that the ellipsis made text uneditable was withdrawn by the human on 2026-09-03 ("A8 is false").

### D. Notes are a second object

**D1. The clamped preview is not the note.** Pain.
Clicking the one-line preview expands the rendered note and does not place a caret. A second click, or Shift+Enter from the title, opens a textarea of source. Workflowy: one click, caret where you pressed.
*State: Phase 2.* The source swap is allowed (A3); the extra click is not.

### E. Only some pages are the outline

**E1. Agenda is not editable.** Pain.
`/agenda` drew a row three weeks late. Clicking it opened nothing. Workflowy: a saved search is still the bullets.
*State: Phase 3.*

**E2. A day page is not editable.** Pain.
`/d/2026-08-10` lists the same node plus a daily-note bullet. The dated rows are scenery.
*State: Phase 3.*

**E3. Zoom is a different page, and had no keys.** Friction.
Glyph click navigates to `/#id` with new chrome. Workflowy zoom is the same document, and has keys.
*State: keys fixed, [#485](https://github.com/juspay/olai/pull/485)* (Alt/⌘+. zooms in, Alt/⌘+, zooms out, Ctrl+Space folds, Ctrl/⌘+↑/↓ fold one way). The page change itself stays: Workflowy's zoom is also a URL change with breadcrumbs. The one remaining difference is that the caret does not survive the zoom; nobody has asked for that yet.

**E4. Done is hidden by default, and had no key.** Friction.
Completing a row can vanish it under the caret. Workflowy keeps completed items until you hide them.
*State: key fixed, [#485](https://github.com/juspay/olai/pull/485)* (Ctrl/⌘+O flips the page's done-hidden). "Stay visible for the row you just completed" is *Phase 3, item 2*, and can ship on its own.

### F. The chord table

| Workflowy | Olai, 2026-09-02 | Olai, 2026-09-03 |
|---|---|---|
| Ctrl/⌘+Shift+Backspace deletes to Trash | unbound; ••• → Move to Trash behind a confirm | unchanged. The human's 2026-08-11 delete ruling still stands; #485 kept the chord out deliberately. Open question 1. |
| Alt/⌘+. and , zoom | unbound | bound, #485 |
| Ctrl+Space, Ctrl/⌘+↑/↓ fold | unbound | bound, #485 |
| Escape = search | Escape = drop the draft | unchanged (A5) |
| Ctrl/⌘+O show/hide completed | a Visible/Hidden toggle on the page | bound, #485 |
| Ctrl/⌘+B/I/U format | unbound | unbound; Phase 1 PR 3 |
| Alt+Shift+↑/↓ move (Win/Linux); ⌘⇧↑/↓ (Mac) | Alt+Shift on every platform | ⌘⇧ added on a Mac, Alt+Shift kept everywhere, #485 |

### G. Phone

Same click-to-input mode. Burger for the directory. No hover •••; long-press is undocumented on screen. Titles ellipsize harder. Workflowy's mobile is a native app with Quick Add; olai's PWA is the desktop editor at 390px. Nothing here is in a phase yet.

## The plan: three phases

Phase 1 makes the current editor stop lying. Phase 2 lets the caret walk the tree. Phase 3 brings the query pages along. Phase 1 finishes first; Phases 2 and 3 are independent of each other after the 2026-09-03 recut, and each is small PRs, not a monolith.

### Phase 1: make the input honest (Layer A)

Stay on `<input>`. Stop throwing information away. Eight steps were proposed; six were built or are being built, two were struck by the human.

| Step | What | Problem | State |
|---|---|---|---|
| 1 | Caret-at-click: measure the click's offset into the rendered title and open the input there | A2 | **shipped**, PR 1 |
| 2 | Letters are not a mode: a bare keystroke opens the last-focused or first visible row | A1 | **rejected** by the human, 2026-09-03: "it is silly" |
| 3 | Enter at column 0 inserts a draft above | B2 | **shipped**, PR 2 |
| 4 | Several local drafts on screen at once | B1, B5 | **shipped**, PR 2 |
| 5 | Paste parser: a clipboard with newlines or leading tabs becomes one atomic write with `children`, never a flattened line | C1 | **open**, PR 3 |
| 6 | Format keys: Ctrl/⌘+B and I wrap the selection in `**` and `*`, and unwrap it when it is already wrapped. No U: the title pipeline drops raw HTML and its sanitiser has no `u`, and markdown has no underline | C2 | **open**, PR 3 |
| 7 | Bind the missing navigation keys: zoom, fold, Ctrl/⌘+O, ⌘⇧ move on a Mac | E3, E4, F | **shipped**, PR 4 |
| 8 | Wrap titles while editing | C3 | **withdrawn as false** by the human, 2026-09-03; the edit face never truncates |

**PR cut**

| PR | Steps | Gate | State |
|---|---|---|---|
| 1 | 1 | Playwright: click at 8px, caret is 0 or 1, not `length` | **merged** 2026-09-02, [#475](https://github.com/juspay/olai/pull/475), the human's own |
| 2 | 3 + 4 | Enter Enter Enter yields three ghosts; Enter at 0 does not teleport to the subtree floor | **merged** 2026-09-02, [#477](https://github.com/juspay/olai/pull/477), grok authored, claude-opus reviewed, approved by the human |
| 3 | 5 + 6 | The four-line beer/party paste becomes three nested rows in one write, and a refused paste lands nothing; Ctrl+B wraps `**` and unwraps it again | **open**. Not filed on the roadmap, not dispatched. The last of Phase 1. |

**What PR 3 has to decide before it is written.** The ops layer's `add_node` already takes a nested `children` tree, three deep, as one validated atomic write (`packages/format/src/writing.ts`, `AddRequest`). The browser's edit wire does not: `packages/surface/src/edit.ts`'s `add` arm is `{ at, title }`, one row per `edit.apply`, and the surface header narrows the wire on purpose. A paste built on that wire would be N sequential writes, stopping at the first refusal with the rest half-landed, which is exactly what `children` exists to prevent. So PR 3 widens the wire: `add` gains an optional `children`, and the server's `addRequest` forwards it. Small in code; the argument that the wire may carry it belongs in the PR body.

The parser also has to say what it does with: two-space and four-space indents; `- ` and `* ` bullet prefixes (Workflowy's own export is `- ` with two-space indent); a multi-line paste into the middle of an existing title (first line joins the title at the caret, the rest become siblings below); and a single line, which stays an ordinary paste.
| 4 | 7 | The chord table above, minus delete and bullet types; each chord an e2e seen red first | **merged** 2026-09-03, [#485](https://github.com/juspay/olai/pull/485), grok authored, claude-opus reviewed, overnight run |

Deferrals the shipped PRs left, all accepted by the human:

- **#477:** titling a middle blank of a before-skeleton is not order-pinned; titling the last (nearest) blank keeps order. Accepted with the PR's approval.
- **#485:** no delete chord (the ruling above). Alt+Shift+Enter stays unclaimed. Ctrl+Space on a Mac shares a key with the OS input-source switch when that is on; the gutter triangle is the fallback. The last two were parked in the Inbox overnight and the PR merged on the morning's word.

Phase 1 closes when PR 3 lands. It does not make olai feel like Workflowy. It makes the current editor stop lying about the click, the paste and the blank line.

### Phase 2: the caret walks the tree (Layer B)

*Not started. Needs the human's word. Recut 2026-09-03; the original shape is in the assessment below.*

**What is already true, and the plan need not build.** There is one live draft and one `<input>` at a time (the single draft signal in `editing.tsx`), so "one editing surface, not 500 inputs" is the editor as it stands. Bare Up/Down already close one input and open the next row's, unconditionally, with a round trip between. The caret-follow after a structural write (`settle` and `follow` in `editing.tsx`) waits on the published frame and puts the caret back. Phase 2 is an extension of that editor, not a replacement for it.

**What to build, as four PRs in Phase 1's style**

| PR | What | Problem | Gate |
|---|---|---|---|
| 5 | **Left at column 0 and Right at end of line cross rows.** Bare ArrowLeft/ArrowRight are unclaimed in `keys.ts`; two clauses there and two actions reusing `step()` with a caret offset (`opened()` already takes one). Up/Down keep the caret's column rather than landing at the end | A4 | Left at 0 lands at the end of the row above; Right at end lands at 0 of the row below; Down from column 5 lands at column 5 |
| 6 | **A draft is a row.** A parked or live empty draft takes Tab, Shift+Tab and Alt+Shift+↑/↓ as local re-anchoring (its `before`/`after`/`under` anchor changes; nothing is written), and draws a bullet, not a hollow placeholder | B5 | Enter, Tab, type, Enter writes a child; the skeleton keeps its shape when titled |
| 7 | **A click on a note preview places the caret at that offset**, using the measurement #475 built for titles. One click, not two. Shift+Enter stays | D1 | Click at 40% of a note's preview: the textarea opens with the caret there |
| 8 | **Split does not teleport.** A placement question first: today `split_node` places the tail as the next *sibling*, which for an expanded parent is after its whole subtree. Workflowy's answer, per its own blog, depends on the fold: an *expanded* parent's tail becomes its *first child*; a *collapsed* parent's tail is the next sibling, after the subtree. If the human takes that placement, the teleport disappears with no optimistic UI, because an expanded parent's first child is the very next line on screen and a collapsed one's next sibling is too. If not, the fallback is scrolling the new row into view with the caret in it | B3 | Split a section title in the middle: the caret's new row is on screen, adjacent to the old one |

**Kept from the original plan**

- **The verbatim-string rule.** The field is one line of source. The improvement is lifetime and placement, not `contenteditable`.
- **What not to build:** a `contenteditable` of the whole page (the 2026-08-09 argument holds); optimistic marks or titles on disk; last-write-wins; blank records on disk.

**Dropped from the original plan, and why**

- **"The first character typed into a row the caret is standing on opens the field."** That needs a state the editor does not have: a focused row with no input open. Every chord from #485 dispatches from the input's own key handler and reads the draft; the page walk returns nothing when no field is open; the only caret-less row identity is the selection layer, which is built to be mutually exclusive with the caret ("a caret or a pick, never both"). Manufacturing the state means a second window-level key listener contending with the selection's, and "one caret" stops being true by shape. And once PR 5 lands, arrowing into a row opens its input, so there is never a row under the caret without a field. The bullet asked for a structural change to reach a state PR 5 makes unreachable.
- **"Dropped if still empty once the caret has been elsewhere and idle."** #477 shipped park-until-page-close, on purpose. A timer that eats a blank while you fetch coffee is worse than a blank that waits.
- **A local draft of position for Tab.** Tab lands in ~110ms and the drive marked it fine. The only real teleport is split (B3), and PR 8 treats that as a placement question, which is cheaper and does not reopen the optimistic-position door the 2026-08 design closed.

**The test.** A person who has used Workflowy for a decade sits down, clicks a row, types into it, hits Enter three times, Tab, pastes a shopping list, clicks a typo in the middle of a long bullet, arrows left off the start of a line, clicks into a note, and never sees a placeholder, a refusal, a source swap they did not ask for, or a caret at the wrong end. If any of those happen, Phase 2 is not done.

### Phase 3: every surface is the outline (Layer C)

*Not started. Recut 2026-09-03. Nothing here is gated on Phase 2 any more.*

**The door the original plan named does not exist.** It said Phase 2's caret would be "about a node id, not a row index", and that this would let the same editor open on an agenda row. The editor is keyed on `Row.key`, a *place*: the chain of ids down the page, deliberately not a node id, so a node reached through two mirrors is two rows. `createEditor` takes the page's rows, its collapsed set and its frame counter; the walk, the follow and the step all move through the flattened tree; an `Anchor` places a new row beside a sibling in that tree. `Editable`, which mounts the editor, exists only on outline and zoomed-node pages, and `useEditor()` throws anywhere else. Day and agenda rows are query entries drawn by `DayNode`, which is `NodeLine` with `onEdit` left off. The missing prop is not the cost; the missing tree is.

**What to build**

1. **Agenda and day rows take the two verbs a hand reaches for there: complete and retitle.** Not the tree editor. A place-less single-field editor on `DayNode`: click the title, get the `<input>` at the clicked offset, Enter commits `set_title`, Escape drops, Ctrl/⌘+Enter completes, nothing else is bound (no Enter-adds, no Tab, no Up/Down). The ops are the same ones the tree sends and the query redraws around the row; if the row leaves the query (completed with hide-done on) the field closes. This satisfies E1 and E2's ask ("tick it, retitle it") without a caret in a query, and needs nothing from Phase 2. The day page's daily-note bullet is unchanged.
2. **Done stays visible for the row you just completed** until you leave the page, with Ctrl/⌘+O as the global. The landing-reveals-done PR already does "revealed, for the visit" for a row you landed on; this is the same reveal for a row you just finished. Independent of everything else here.
3. **Tab, Enter-adds and arrows on an agenda row** stay out until someone shows a use for them. Workflowy allows them on a search result; nobody in the drive missed them.

**Dropped:** *zoom as focus*. The original plan wanted #485's zoom keys to scroll and collapse in place rather than navigate to `/#id`. Workflowy's zoom also changes the URL and draws breadcrumbs; what it keeps that olai does not is the caret, which #485 documents as "the caret stays behind". Re-deciding zoom a day after shipping it is churn. If anyone wants the caret to survive a zoom, that is a PR the size of Phase 2's PR 5.

## Assessment of the plan, 2026-09-03

The plan was written by Grok from the browser drive. The diagnosis is right and Phase 1 was well cut: four small PRs, each putting a key or a click onto behaviour that already existed, each with an e2e gate seen red first, three shipped in two days. The rest was checked against the client code on 2026-09-03 and did not all hold:

- **The paste parser's atomicity was assumed, not available.** `add_node` takes `children`; the browser's edit wire does not. Written as proposed, the paste would have been N writes with a half-landed failure mode. PR 3 now says the wire widens, and lists the clipboard shapes the parser has to answer.
- **Phase 2 described the existing editor as if it were new.** One surface, one live input, Up/Down walking rows: all shipped. The novel parts were four small things (edge-crossing arrows, real-bullet drafts, note click, split placement) and one structural thing (a focused row with no input) that the arrows make unnecessary. The monolith was gated on "the human's word" as one product change; the recut is four PRs that each ship alone.
- **Phase 2 contradicted #477** (drafts dropped on an idle timer versus drafts parked until the page closes) and proposed optimistic position for a Tab the drive itself called fine.
- **Phase 3's justification was false.** The editor is not keyed on node ids, and Phase 2 as recut does not change that. The recut says what is structural (a tree-keyed editor on a query page) and offers the two verbs that matter as a small place-less editor instead, ungated.
- **Zoom-as-focus misread Workflowy**, whose zoom is also a URL change with breadcrumbs.
- **Underline was never possible.** The title pipeline drops raw HTML and its sanitiser allowlist has no `u`; step 6 no longer hedges on it.
- **Escape (A5) and the phone (G) are still unplanned.** Both are marked open rather than hidden.

Two things the recut could not settle are questions 5 and 6 below: the wire widening and the split placement.

## Rulings log

| Date | Ruling |
|---|---|
| 2026-08-11 | No delete key. Still standing; see open question 1. |
| 2026-09-02 | PR 1 (#475) shipped by the human. PR 2 (#477) dispatched to grok, approved "PR 477 is approved" at 16:41 ET and merged. PR 4 chosen for the overnight run. |
| 2026-09-03 | Step 2 rejected: "I reject A2, it is silly." Step 8 withdrawn: "A8 is false." #485 merged on the morning's word with its two Inbox deferrals accepted. |

## Open questions for the human

1. **Delete key.** Deferred 2026-08-11. The ••• menu already puts a subtree in Trash behind a confirm. Binding Ctrl/⌘+Shift+Backspace to that same confirm is the Workflowy chord without inventing a shredder. A bulk chord is the dangerous one. #485 left it out on this ruling.
2. **Empty titles on disk.** Phases 1 and 2 keep the format. If that ever changes, sketching gets simpler and git gets noisier. Recommendation: do not change it.
3. **Title wrap on the read face.** A reading question only, since the edit face never truncates (C3). Does a long row wrap when you are looking at it, or keep the ellipsis the quiet-outline ruling chose?
4. **Should agenda and day pages take complete and retitle.** Phase 3 item 1 argues yes, through a place-less single-field editor that binds nothing structural. The original ban was about a caret in a *query* walking onto a neighbour; a field that cannot walk does not trip it. If the ban is load-bearing for another reason, Phase 3 becomes "click through to the home outline" and the feel stays split.
5. **May the browser's edit wire widen for paste?** PR 3 needs `add` to carry `children` so a pasted tree is one atomic write. `packages/surface/src/edit.ts` narrows the wire on purpose; the paste is the first hand-driven write that needs the wider one.
6. **Where does the tail of a split parent go?** Today: next sibling, after the whole subtree, which is the teleport in B3. Workflowy: first child when the parent is expanded, next sibling when it is collapsed, so the new line is always the next line on screen. Taking Workflowy's placement closes B3 with no UI work. `split_node`'s contract ("placed immediately after it") would gain the expanded case, and the browser would have to send the fold state or choose the placement itself.
7. **Phases 2 and 3.** Neither is dispatched until the human says so. Each is cut so that any single PR can go alone.

## What this is not

A request to become Workflowy. Olai is files, git, agents, marks with instants, mirrors that are placements, an agenda that is owed work. Those are the product. The ask is that hands that already know an outliner are not fighting a viewer.
