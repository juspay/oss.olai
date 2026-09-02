# Editing UX vs Workflowy — driven 2026-09-02

Status: assessment + proposal. Not a rewrite of [editing-web.md](editing-web.md) (that file is the 2026-08 research the editor was *built from*). This is what a Workflowy hand actually hits in the browser today, after that editor shipped.

Method: `just serve` on the `good` fixture, Playwright driving `http://127.0.0.1:7788` the way a person types — click, type, Enter, Tab, paste, arrows, chords, agenda, day, phone viewport. Screenshots under `/tmp/olai-uireview-shots/`.

The 2026-08-12 inventory already listed missing *features* (paste-as-tree, format keys, delete key, bullet types). The thing that sucks is not the missing menu entries. It is that **the outline is a viewer you switch into an editor**, and Workflowy is **an editor you can view**.

## The diagnosis

Three design choices, all deliberate, compose into the feel:

1. **A title is an `<input>`**, mounted on click, unmounted on blur. `RowEditor.tsx`: *"Absent is the end of the text, which is what a click on a title means."* `takeCaret` focuses and `setSelectionRange(length, length)`.
2. **Nothing is optimistic.** Structural keys are intents; the row moves when the file says so. Empty titles are illegal on disk, so a new row is a ghost draft until it has words.
3. **Only outline and zoom pages are editable.** Agenda and day are query views. Notes are a second field behind a ¶ / clamped preview.

Workflowy's model, from the help pages driven against: click any character, the caret is there; Enter always makes a real bullet, including empty; Tab is the same frame; paste of indented text is a tree; notes are already under the bullet; zoom/collapse/search are keys; every filtered view is still the same document.

Olai copied Workflowy's *chords* (Tab, Shift+Tab, Ctrl+Enter, Shift+Enter, (( , ! , #) and not its *continuity*.

## What is already fine

Do not "fix" these — they match, or they are olai on purpose:

- Tab / Shift+Tab indent. Measured ~110ms to land. Acceptable.
- Ctrl/Cmd+Enter completes. Glyph click zooms (Workflowy too).
- `((` mirrors, `!` dates, `#`/`@` tags, ⌘⇧D duplicate, ⌘⇧M move-to, drag-a-bullet, the five multi-select gestures, ••• menu.
- Backspace at column 0 merges (round trip, but it merges).
- Split at caret exists and keeps children on the first half.
- No last-write-wins, no `#copy` tag, no completing-a-parent-completes-children. Keep those refusals.

## Problems, by cause

Severity: **blocker** = a Workflowy hand cannot do the thing at all. **pain** = they can, but the feel is wrong every time. **friction** = extra click, missing chord, or a surface that is read-only. **gap** = listed in 2026-08-12 and still missing.

### A. There is a mode. Workflowy has none.

**A1. Typing does nothing until you click a title. BLOCKER.**
Focus is `BODY`. There is no caret in the outline. Workflowy: the caret is always in some bullet; you just type.

**A2. Click does not place the caret. BLOCKER.**
Clicked 8px from the left of `choose the handles` (18 characters). Caret landed at 18/18. Clicked at 40% of `kitchen remodel #home`. Caret landed at end. The span is torn down and an `<input>` is mounted; the click's X is thrown away. `takeCaret` then puts the caret at `value.length`.

This is the single most important defect. A Workflowy hand clicks the typo they saw. Olai opens an editor at the other end of the line.

**A3. The read face and the edit face are different documents. PAIN.**
`#home` is a coloured chip until click, then raw `#home` in an input. Shift+Enter on a note shows `**walnut**` and `*birch*` as source; the preview one click earlier rendered them bold/italic. The row washes. Tags restyle. The outline around you is still the read face, so you are the only row in "source mode."

**A4. Left/Right at the ends of a line do not leave the row. PAIN.**
Home, then Left: still `pick the knobs` at 0. Workflowy treats the outline as one text stream: Left at column 0 is the end of the bullet above. Olai's arrows between rows are Up/Down only, and each hop closes one input and opens another *at the end*.

**A5. Escape throws the draft away. FRICTION.**
Typed ` xyz` on `garden`, Escape: title is `garden #outdoors` again. Workflowy's Escape is search. Letters already belong to the document, so there is nothing to discard.

### B. Blank bullets cannot exist. Outlining is sketching blanks.

**B1. Enter Enter Enter does not make a list. BLOCKER.**
Five Enters on `the compost heap`: one ghost, placeholder *"a new line — type it, and Enter makes the next one."* Empty drafts write nothing (correct for git). The cost is you cannot lay out a skeleton and fill it in. That *is* outlining.

**B2. Enter at column 0 does not insert above. PAIN.**
Caret at 0 on a titled row, Enter: a ghost sibling appears *after the entire subtree*, at the bottom of the page. Documented: empty titles are illegal, so this is `add` not insert-above. Workflowy: Enter at column 0 puts a blank bullet above and keeps your words where they were. That is how you make space.

**B3. Split of a parent teleports the caret. PAIN.**
Split `kitchen remodel #home — extra` in the middle: first half stays the section (children intact), second half `odel #home — extra` appears as a new top-level *below the whole tree*. You were typing at the top; you are now at the bottom. Semantics match Workflowy (children stay with the first half). The teleport is the no-optimistic-UI + "sibling after subtree" combination, made worse because the new half is an input on a washed row a screen away.

**B4. Clearing a title is refused. PAIN.**
Select-all, Backspace, Enter: red *"a node needs a title"* under an empty input. The row cannot exist as a blank. Workflowy: blank bullets are first-class.

**B5. The new-row ghost is not a bullet. PAIN.**
Hollow circle, muted placeholder, no Tab until it has a title, click-away destroys it. It often renders far from the key that created it.

### C. An `<input>` cannot be a tree.

**C1. Paste of a nested list is one line. BLOCKER.**
Clipboard:

```
buy beer
\tgarden party
\t\tinvite neighbours
backup plan
```

Landed as the ghost *"buy beer  garden party    invite neighbours backup plan"*. Newlines cannot live in an `<input>`. Node count unchanged. This is the capture path Workflowy is loved for, still MISSING from the 2026-08-12 inventory, and now confirmed in the browser as a flatten, not a no-op.

**C2. Ctrl+B / Ctrl+I do nothing. GAP.**
Select all, Ctrl+B: value still `the compost heap`. No markdown wrap, no toolbar. Titles already *render* `**bold**` (#84). The keys should write the markdown the renderer already understands.

**C3. Long titles ellipsize. PAIN.**
A 1113px sentence in an 882px cell, `overflow: hidden; text-overflow: ellipsis; white-space: nowrap`. On a 390px phone, `order the new cabinets` is `order the...`. The quiet-outline ruling (a row is a line) is why. Workflowy wraps. You cannot edit what you cannot see, and click-to-end (A2) on an ellipsized title is strictly worse.

### D. Notes are a second object.

**D1. The clamped preview is not the note. PAIN.**
`order the new cabinets` shows *"Two ways to go:"* under the title (cozy density). Clicking that line expands the rendered note (walnut bold, see-links) and does **not** put a caret in it. Second click, or Shift+Enter from the title, opens a textarea of source. Workflowy: the note is already under the bullet, one click, caret where you pressed, no source swap.

The 2026-08-11 revision already rejected "Shift+Enter opens an ugly textarea" and landed in-place styling. The remaining gap is the extra click and the source swap, both forced by "rendered markdown is a reading surface we must not destroy on the first press."

### E. Only some pages are the outline.

**E1. Agenda is not editable. PAIN.**
`/agenda` drew `order the new cabinets`, 3 weeks late. Clicking the title opened no editor. You can see the work; you cannot tick it, retitle it, or Tab it. Workflowy: a saved search is still the bullets.

**E2. A day page is not editable. PAIN.**
`/d/2026-08-10` lists the same node, plus a daily-note bullet. Click opened no editor. `+ day note` mints a `.md`. The dated outline rows are scenery.

**E3. Zoom is a different page. FRICTION.**
Glyph click on kitchen → `/#kitchen`. Glyph click on garden → `/#garden`. Breadcrumbs, new chrome, the rest of the file gone. Workflowy zoom is the same document; Alt/Cmd+. / , and sibling-page keys stay in it. Olai has no zoom keys (Alt+. / Alt+→ / Cmd+, did nothing) and no collapse keys (Ctrl+Space, Ctrl+↑/↓ unbound).

**E4. Done is hidden by default. FRICTION.**
`take out the old counters` is gone. Parent still says 1/2. The page has Visible/Hidden, not Ctrl+O. Completing a row can vanish it under the caret (hide-done-scope, shipped). Workflowy keeps completed items until you hide them.

### F. Missing chords the hands already know.

| Workflowy | Olai today |
|---|---|
| Ctrl/Cmd+Shift+Backspace deletes to Trash | unbound. ••• → Move to Trash, behind a confirm. Human 2026-08-11 still open. |
| Alt/Cmd+. and , zoom | unbound. Click the glyph. |
| Ctrl+Space, Ctrl+↑/↓ fold | unbound. Click the triangle. |
| Escape = search | Escape = abandon draft |
| Ctrl/Cmd+O show/hide completed | Visible/Hidden toggle on the page |
| Ctrl/Cmd+B/I/U format | unbound |
| Alt+Shift+↑/↓ move (Win/Linux); Cmd+Shift+↑/↓ (Mac) | Alt+Shift on every platform |

### G. Phone

Same click-to-input mode. Burger for the directory. No hover ••• (long-press, undocumented on screen). Titles ellipsize harder. Workflowy's mobile is a native app with Quick Add; olai's PWA is the desktop editor at 390px.

## Root constraint, said once

The format will not hold an empty title. The wire will not echo a write. The title editor will not be `contenteditable` because a title is one verbatim string and a live frame would fight the caret.

All three are still right for *git* and *agents*. They are the wrong stack for *hands*. The proposal below keeps the first two (empty-on-disk stays illegal; the file stays the truth) and replaces the third.

## Proposal

Three layers. A is a week. B is the actual product. C is the rest of the app catching up. Do not start C before B; do not skip A.

### Layer A — make the input honest

Stay on `<input>`. Stop throwing information away.

1. **Caret-at-click.** On `clickTitle`, measure the glyph offset into the rendered title (a hidden copy of the text, or `document.caretPositionFromPoint` on the span *before* unmounting it) and pass that index into `takeCaret`'s `at`. The comment that says "a click on a title means the end" is the bug. This one change removes the worst Workflowy break.
2. **Click focuses; letters are not a mode.** If no draft is open and the user types a character with the outline focused, open the last-focused row (or the first visible row) at its end and insert. Never swallow keys into `BODY`.
3. **Enter at column 0 inserts a draft *above*.** Still a draft, still writes nothing until it has a title. The current "add after the subtree" reading is what you want at *end* of line, not at *start*.
4. **Several local drafts.** One caret, but abandoned empty siblings may remain on screen as drafts until the page closes — or, cheaper, Enter on an empty draft opens the *next* empty draft without collapsing the first. Disk still sees nothing. This is how Enter Enter Enter becomes a skeleton without weakening `a node needs a title`.
5. **Paste parser.** On paste into a title (or a new-row draft), if the clipboard has newlines or leading tabs, do not put it in the `<input>`. Parse a tab-indented outline and send one `add_node` with `children` (the op already takes a tree). Flattening to one line is the worst possible answer.
6. **Format keys write markdown.** Ctrl/Cmd+B/I/U wrap the selection in `**` / `*` / nothing-olai-renders-as-underline (skip U, or wrap `<u>` if the title renderer grows it). No toolbar required for A.
7. **Bind the missing navigation keys.** Zoom in/out, fold, Ctrl+O for the page's done-hidden, platform-correct move-among-siblings on Mac (Cmd+Shift+↑/↓). Delete stays the human's; until that ruling, at least bind Ctrl/Cmd+Shift+Backspace to the same confirm the menu already asks.
8. **Wrap titles, or wrap while editing.** Ellipsis on the read face is a reading choice. Ellipsis on the *edit* face is uneditable text. At minimum the open `<input>` wraps (or the row grows). The quiet-outline ruling can stay for unread rows.

Layer A does not make olai feel like Workflowy. It makes the current editor stop lying about the click, the paste, and the blank line.

### Layer B — the caret is the outline

Replace "click a title, mount an input" with a continuous caret over the tree.

**What to build**

- Every visible title is already a field. Not 500 `<input>`s: one editing surface whose *value* is the row the caret is in, drawn in place of that row's title, with neighbouring rows still the read face — *but the caret can walk into them without a click*. Up/Down, Left-at-0, Right-at-end move it. The first character you type in a rendered row opens the field *at that character*.
- Keep the verbatim-string rule. The field is still one line of source. The improvement is lifetime and placement, not contenteditable HTML.
- Notes: clicking the preview puts the caret in the note at that offset (source swap allowed; the extra click is not). Shift+Enter stays the keyboard door.
- New rows are real rows in the tree widget *locally* (a draft slot with a bullet, indent, Tab, arrows), committed when they have a title, dropped if still empty when the caret has been elsewhere and idle. The ghost placeholder goes away.
- Split/indent still wait on the file. The caret-follow primitive that `editing.tsx` already owns is the whole of the wait; the row must not *look* like it hasn't moved. A 100ms local indent, confirmed by the file, is not optimistic UI of the kind the design forbade — the forbidden thing was two tabs disagreeing about what landed. A caret sliding under the parent, then snapping if the write is refused, is a draft of *position*, which we already do for *text*.

**What not to build**

- A contenteditable of the whole page. Paste-as-HTML, caret vs live frames, innerHTML → title string: the 2026-08-09 argument still holds.
- Optimistic marks, optimistic titles on disk, last-write-wins.
- Blank records on disk. Local blank drafts only.

**The test**

A person who has used Workflowy for a decade sits down, does not click, types, hits Enter three times, Tab, paste a shopping list, clicks a typo in the middle of a long bullet, and never sees a placeholder, a refusal, a source-swap they didn't ask for, or a caret at the wrong end. If any of those happen, B is not done.

### Layer C — every surface is the outline

Only after B:

- **Agenda and day pages edit the node they draw.** The 2026-08 ruling that a day is a query, not a tree, is why the keyboard loop was banned there (a caret in a query would re-place onto whoever matched next). B's caret is about a *node id*, not a row index in a tree. That is the door: open the same field on an agenda row, send the same ops, let the query redraw around it. If the row leaves the query (you completed it, hide-done is on), the caret follows to the node's home or closes — it does not jump onto a neighbour it was never in.
- **Zoom keys** over B's caret (already in A as bindings; in C they should not be a navigation to `/#id` unless the user wants a page). Prefer zoom-as-focus (scroll + collapse the world above) and keep `/#id` for the address bar / pins.
- **Done-hidden** defaults to Workflowy's "stay visible" for the row you just completed, or Ctrl+O as the global. Vanishing the row under the caret is a feel bug even if it is the preference's right answer for *reading*.

## Suggested PR cut

| PR | Layer | What lands | Gate |
|---|---|---|---|
| 1 | A1+A2 | caret-at-click; type-into-focused-outline | Playwright: click at 8px, caret is 0 or 1, not `length`. Type with no prior click inserts into a row. |
| 2 | A3+A5 | Enter at column 0 inserts a draft above; multiple empty drafts on screen | Enter Enter Enter yields three ghosts. Enter at 0 does not teleport to the subtree floor. |
| 3 | A paste + format keys | newline/tab clipboard → `add_node` tree; Ctrl+B wraps `**` | The beer/party paste becomes three nested rows. |
| 4 | A keys | zoom, fold, Ctrl+O, Mac move chord, delete-or-confirm | The Workflowy chord table, minus bullet-types. |
| 5 | B | continuous caret, local blank slots, note-click-is-caret | The test in Layer B. |
| 6 | C | agenda/day edit by node id | Click `order the new cabinets` on `/agenda`, type, it writes. |

PR 1 is the one that should have shipped with `self-edit`. It is also the only one that can land without a design argument.

## Open questions for the human

These were already open; driving the browser did not close them:

1. **Delete key.** 2026-08-11 deferred. The menu already puts a subtree in Trash behind a confirm. Binding Ctrl/Cmd+Shift+Backspace to that confirm is the Workflowy chord without inventing a shredder. A bulk chord is still the dangerous one.
2. **Empty titles on disk.** Layer A/B keep the format. If that ever changes, sketching gets simpler and git gets noisier. I would not change it.
3. **Title wrap vs the quiet outline.** Ellipsis is a reading choice that fights editing. Wrap-while-editing is the compromise; wrap-always is the Workflowy one.
4. **Should agenda/day be editors.** Layer C argues yes, by node id. The original ban was about a caret in a *query*. If that ban is load-bearing for another reason, C becomes "click through to the home outline" and the feel stays split.

## What this is not

A request to become Workflowy. Olai is files, git, agents, marks with instants, mirrors that are placements, an agenda that is owed work. Those are the product. The ask is that the *hands* that already know an outliner are not fighting a viewer.
