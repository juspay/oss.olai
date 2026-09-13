# Drag rows, files and transcript rows into a conversation, and transcript rows into an outline

Implementation plan for one PR on branch `issue-584`, resolving [#584](https://github.com/juspay/olai/issues/584). Written for an agent that has not seen the discussion; every claim below about the current code was checked against this tree at `e19da09de`. Where a fact still has to be found out, the item says so and says what to do in either case. Section 8 lists the decisions the human is asked to approve before work starts.

## 1. What changes for a person

Today a row reaches a conversation only through **Ask agent** in its `•••` menu, which targets the nearest agent *above* the row (`packages/plugins/chat/src/browser/verbs.tsx:16-53`) and refuses otherwise. After this PR the target is wherever you let go:

1. **Drag a bullet onto a conversation and the row is armed there.** The same press-and-travel a row move is (`docs/editing.md` *Dragging a row*): carry the bullet over the fold or page of any conversation — in this file, another file, the other pane — and the panel lights the way it lights for a file, saying *drop to ask about it*. Let go and the row is a chip above the box, exactly as `@` completion arms one. A pick of several rows arms all of them, in pick order. The row does not move: a conversation is not a gap in an outline.
2. **Drag a transcript row onto another conversation's composer and it lands as quoted text.** Every row that has words — the agent's answer, your own message, a tool call with its result, a diff box — grows a small grip at its left edge on hover. Carry it to a composer and its text is written at the caret as a Markdown quote (`> ` on every line, a blank line after). This is how one agent's answer is handed to another; dropping it back on its own composer quotes it too. On a phone, hold the row and it lifts, as a bullet does; a finger that moves first scrolls.
3. **Drag a file from the sidebar onto a composer and its path is written into the sentence**, spelled exactly as `@` completion spells it (`read @notes/cabinets.md `). Pressing a file row without travelling still opens it. On a phone, hold the row and it lifts; the drawer closes so the page you were reading is the target (5.6).
4. **Drag a transcript row between two outline rows and it becomes a node there.** The same drop line a row move draws says which gap and how far in. The row's first line is the title and the rest is the note; ⌘Z takes it back like any other add.
5. **Ask agent leaves the row menu.** Its keyboard equivalent is `@` in the composer, which already names any node. **Start an agent session** stays. The palette's `>` keeps sending to the agent above the focused row and keeps its two refusals.
6. **Unchanged:** dropping OS files on the panel attaches them; row moves within and across panes; the cross-file refusal for row *moves* (a conversation is not an outline, so a row from `lanes.olai` lands on an `orchestrator.olai` agent's composer without one).

## 2. Non-goals

- No HTML5 drag-and-drop for the new gestures (section 3, D1). OS file drops stay on the HTML5 path they use today (`packages/plugins/chat/src/browser/chat/DropTarget.tsx`).
- The zoomed plain node's dashed composer (*ask about <title>…*, `packages/plugins/chat/src/browser/agents/Page.tsx:115`) is not a landing. It has no armed strip and its first send starts an agent (`Page.tsx:65-81`), so a node dropped there has nothing to be armed in yet. Carrying anything over it lights nothing. Noted in section 9.
- No attribution line on a quote (*from <agent>:*). The person types the framing; the quote is the words.
- No change to the ACP wire, the `context` field of a send (`packages/plugins/chat/src/wire.ts:141-154`), the server's resolution of an armed id (`packages/plugins/chat/src/server/context.ts:56-93`), the `add`/`place` ops, or the `@` completion.
- No drop of a file row into an outline, and no drop of an outline row onto a document page. Neither lights anything and release does nothing.

## 3. Decisions, with the reasons

**D1. One gesture system: pointer carries.** The outline row drag is pointer-based on purpose — `@olai/web`'s `drag()` (`packages/web/src/client/pointer.ts:117`) gives it the 4px travel threshold, Escape cancellation, edge auto-scroll and the touch long-press, and the bullet suppresses native drag (`packages/plugins/outlines/src/browser/drag/Handle.tsx:61-62`). Making a composer a landing of *that* gesture means the composer must be found by pointer position, not by `dragenter`. So the two new carriers (transcript rows, file rows) use the same `drag()` primitive, and only OS files stay HTML5. What this gives up is a drop from another browser window, which nobody asked for.

**D2. A host-supplied registry of landings, not a slot and not a service of outlines or chat.** Three plugins carry (outlines, chat, files) and two receive (chat's composers, outlines' pages); chat must not wait on outlines to take a file, and outlines must not wait on chat to move a row. The one thing every browser plugin can name without waiting is what `openApp` supplies itself — `Edits` is supplied that way for exactly this reason (`packages/plugin-api/src/browser.ts:496-507`, `:806-809`). So `openApp` supplies one `Landings` table per app; receivers register for a component lifetime; a carrier snapshots the table when it lifts. Absent providers are simply fewer receivers, which is the optional-availability shape `docs/architecture/cordis.md` asks for.

**D2a. The host knows only that a payload has a kind; each carrier owns its payload's shape.** `plugin-api` would otherwise be told what an outline node or a served file is, which is feature vocabulary leaking upward. So the table's `Carried` is `{ readonly kind: string }` and nothing more, and each carrying plugin declares its own payload type in a contract door of its own — `olai-plugin-outlines/carry` (`{ kind: "outlines.nodes", ids, file }`), `olai-plugin-chat/carry` (`{ kind: "chat.text", text }`), `olai-plugin-files/carry` (`{ kind: "files.path", path }`) — each with a type guard. A receiver imports the guard it takes from the carrier's door: a static contract, which the Cordis rule allows (`docs/architecture/cordis.md` *Imports are not live dependencies*), and the same shape every plugin already uses for its own slot descriptors. Nothing live crosses; a receiver that does not recognise a kind answers null at lift.

**D3. The receiver owns its indicator and its write.** A composer lights its own overlay (the one `DropTarget.tsx:112` already draws) and arms or writes into its own draft; an outline page plans its own drop line with its own `planDrop` and sends its own `add`. The carrier knows only boxes and a payload. This keeps the drop line, the `Aiming` portal and the write path inside outlines, where they are today.

**D4. Innermost box wins.** A fold's conversation sits inside its outline page's box, and a zoomed page's conversation sits inside that page's. When the pointer is inside more than one receiver, the smallest box takes it; when it is inside none, the outline row drag falls through to today's page aiming.

**D5. What a transcript row carries is the words the row is drawn from**, never the rendered DOM: an agent row its Markdown source (`AgentEntry.text`), a user row its text, a tool row its title followed by its reply (`detail` if present, else the reply JSON the fold prints, `ToolFrame.tsx:447-456`), a diff box its path and the `+`/`-` lines `diffOf` derives (`packages/plugins/chat/src/browser/chat/diff.ts:98`), an outline-diff box its change sentences. Rows with no words — notices, refusals, ask forms — have no grip, and a still-streaming agent row has none until it ends.

**D6. Into an outline: first line is the title, the rest is the note.** A title cannot hold a newline and a node is the one shape an outline has. The text goes in as the `add` op takes it; a sentence starting with a sigil the format reads is the format's business, as it is when typed.

**D7. A carrier that stops mid-carry tells every lit target the pointer has left.** A gesture belongs to its component through `createDrags()` (`pointer.ts:212-216`), so a plugin rebuild under a held row tears the gesture down. That teardown runs the carry's cancel path — `leave()` on the aimed receiver — so no panel stays lit and no drop line stays drawn after its carrier is gone. `drag_live_recovery.feature` already proves the outline half of this; the new carriers get the same scenario.

**D8. Touch carries for transcript rows and sidebar file rows are in this PR.** The same rule the bullet has (`docs/editing.md` *With a finger, hold the bullet first*): hold the row for the platform's long press (`LONG_PRESS_MS`, `packages/web/src/client/longPress.ts:66`), it lifts and follows the thumb; a finger that moves before then scrolls. Transcript rows have no other long-press today, so nothing is displaced. The sidebar on a phone is a drawer, which 5.6 has to settle.

## 4. Ownership after the change (Cordis)

| Thing | Owner | How others reach it |
|---|---|---|
| The `Landings` table: receivers registered, snapshot at lift | the browser host (`openApp`), one per app, supplied like `Edits` | declared `needs: [Landings]` on the consuming component; each consumer keeps its own `heldService` holder (`packages/ui-primitives/src/held.ts`), never a provider's |
| The static contract: `Carried = { kind: string }`, `Receiver`, `Box`, `carry()` helper | `@olai/plugin-api` (`src/carry.ts`) — types, a factory and a pure hit-test; no live state, no feature vocabulary | import, as `Edits`' shapes are imported |
| Each payload's shape and type guard (`outlines.nodes`, `chat.text`, `files.path`) | the carrying plugin, in its own `/carry` contract door | a receiver imports the guard for the kinds it takes: a static contract, never a live value |
| The pointer gesture (`drag`, `TRAVEL_PX`, `createDrags`) | `@olai/web/client/pointer.ts`, as today | static import, as pins and panes import it |
| A composer's receiver: box, overlay, arm / quote / name | chat, registered by the `Conversation` component (`packages/plugins/chat/src/browser/agents/Fold.tsx:38`) on mount, released on cleanup | reached only through the table |
| An outline page's receiver for text: box, drop line, `add` | outlines, registered by `Editable` beside `useFields().join` (`packages/plugins/outlines/src/browser/edit/Editable.tsx:181`) | reached only through the table |
| The outline row carry consulting receivers | outlines' `createDragging` (`drag/dragging.ts:171`), which already owns the gesture | it reads the table it was handed |
| Transcript grips and the text carry | chat, per row (`Row.tsx`, `Diff.tsx`, `OutlineDiff.tsx`) | — |
| Sidebar file row carry | files (`packages/plugins/files/src/Files.tsx` / `fileTree.ts`) | — |
| Armed nodes, drafts, caret | chat's `ConversationUI` per `[agent, session]`, as today (`browser/chat/ui.tsx:12-24`, `armed.ts`, `message-draft.ts`) | a composer receiver writes into its own conversation's UI and nothing else's |
| `>` command and the nearest-ancestor lookup | chat's `verbs.tsx` `target`/`show` and the server's `agentAbove` (`server.ts:566`), unchanged | — |

Removed: the `ask-agent` entry of `rowVerbs` (`verbs.tsx:35-41`). Nothing else is withdrawn; `outline.row.action` keeps the start entries.

Withdrawal: a receiver registration belongs to its component (Solid `onCleanup`) *and* to its activation (the holder is released when the activation stops), so a fold closing, a row being trashed, a pane closing or the chat row being switched off each take that receiver out. A carry in flight keeps only the snapshot; at release the carrier asks the live table whether the receiver is still registered and does nothing if it is not — the same shape `drag_live_recovery.feature` proves for a destination removed under a held drag.

## 5. Find out first

**5.1 Where the contract may live.** `just cordis-deps` is the import fence. Confirm that `@olai/plugin-api` may hold `src/carry.ts` (types + `createLandings()` factory + pure `innermost(boxes, x, y)`) and that chat, outlines and files browser components may name a `Landings` tag from `plugin-api/browser`. If the fence refuses the factory in plugin-api, put the factory in `@olai/ui-primitives` beside `held.ts` and keep only the tag in plugin-api; the supply in `openApp` stays.

**5.2 Coordinates.** `drag()`'s `onPage(x, y)` hands document coordinates (`pointer.ts:58-105`), and outlines measures boxes once at lift in the same system (`drag/lines.ts:114-130`, `dragging.ts:272`). Receivers measure the same way. Check that a fold's box inside a split pane's own scroller is measured in document coordinates too (the pane scrolls, the window does not); if `measureBox` is window-relative there, aim by the element's live `getBoundingClientRect()` converted per move instead of a lift-time box, and say so in the receiver interface.

**5.3 The sidebar row element.** Find the file row in `packages/plugins/files/src` (`Files.tsx`, `fileTree.ts`). It is expected to be an `<a>` like the bullet and the pin; the pin's pattern applies (`packages/plugins/pins/src/browser/Pin.tsx:77-90`: `onPointerDown` grab, `draggable={false}`, `onDragStart` prevented, capture-phase click swallowed after a travelled drag). Confirm the path the row carries is the served path `@` completion writes (`browser/chat/naming.ts:113-144` reads `useServed()`), so both doors spell one path.

**5.4 The reply text of a tool row.** Confirm `ToolEntry.detail` and `reply` are the two the fold prints and that nothing else (terminal output, `progress`) should ride the quote. Terminal output is streamed under the row; leave it out and say so.

**5.6 The phone drawer.** On a phone the sidebar is a drawer `min(22rem, 92vw)` wide over a dimming backdrop that closes it on tap (`packages/plugins/sidebar/src/Sidebar.tsx:21`, `:38`, `:72`), so a file lifted in it has no visible target. Find who owns the drawer's open state and whether it is reachable through a declared service. Preferred: when a held file row lifts, the files row asks the sidebar to close the drawer through that service (optional for files — a `components` entry with its own `needs`, so files without a sidebar service still draws its tree), the backdrop goes with it, and the carry continues over the page the person was reading; the phone scenario proves a drop on a fold there. If no such service exists, sidebar declares one (`sidebar.drawer`, owned by sidebar, `open`/`close`, withdrawn with it) — a small addition, and the only honest way for one plugin to move another's drawer. Only if that proves out of reach does the phone scenario shrink to "a held file row lifts and a release inside the drawer does nothing", and `docs/chat.md` says so.

**5.5 Attribute monopolies.** `packages/web/src/client/claims.test.ts:329` asserts `data-handle` belongs to outlines' drag only. The transcript grip and the file row use their own attributes (`data-grip` in chat, none needed in files beyond a testid); add them to that test's tables rather than widening `data-handle`.

## 6. Work items, in commit order

Commit after each item; `just typecheck-fast-remote` after 6.1, `just e2e-fast-remote` from 6.2 on. Each item names its e2e coverage; write the scenario in the same commit as the behaviour. New feature files follow `packages/tests/README.md` and reuse `packages/tests/support/dragging.ts` (`carry`, `pressBullet`), never `page.dragAndDrop`.

### 6.1 The table

`packages/plugin-api/src/carry.ts`:

```ts
/** What is in the air. The host knows only its kind; the shape is the carrier's (D2a). */
export interface Carried { readonly kind: string }

export interface Box { readonly top: number; readonly left: number; readonly bottom: number; readonly right: number }

export interface Receiver {
  /** Measure yourself for this carry, in document coordinates; null means "not for this kind". */
  readonly lift: (carried: Carried) => Box | null
  /** The pointer is over you: draw your indicator. */
  readonly aim: (carried: Carried, x: number, y: number) => void
  /** The pointer left, or the carry was cancelled: clear it. */
  readonly leave: () => void
  /** Released over you: take it. A string is the refusal to say. */
  readonly drop: (carried: Carried, x: number, y: number) => Promise<string | null>
}

export interface Lifted { readonly receiver: Receiver; readonly box: Box }

export interface Landings {
  readonly register: (receiver: Receiver) => () => void
  readonly lift: (carried: Carried) => ReadonlyArray<Lifted>
  readonly standing: (receiver: Receiver) => boolean
}
export const Landings = serviceTag<Landings>("landings")
export const createLandings = (): Landings => …
/** The innermost of the boxes containing (x, y), else null. Pure. */
export const innermost = (lifted: ReadonlyArray<Lifted>, x: number, y: number): Lifted | null => …
```

Each carrier's door declares its payload and guard, in the same commit:

```ts
// olai-plugin-outlines/carry
export interface CarriedNodes extends Carried { readonly kind: "outlines.nodes"; readonly ids: ReadonlyArray<string>; readonly file: string }
export const carriedNodes = (c: Carried): c is CarriedNodes => c.kind === "outlines.nodes"
// olai-plugin-chat/carry   → CarriedText  { kind: "chat.text"; text }
// olai-plugin-files/carry  → CarriedPath  { kind: "files.path"; path }
```

Supply the table in `openApp` next to `Edits` (`browser.ts:806-809`), with the same comment: no row stands behind it, naming it costs no wait. Add a `carry(from: PointerEvent, carried, landings, { threshold, onEnd? })` helper in the same file that runs `drag()` from `@olai/web` — check 5.1 for whether plugin-api may import `@olai/web/client/pointer`; if not, the helper lives in `@olai/web/client/carry.ts` and each carrier imports it from there — snapshots `landings.lift(carried)` at start, calls `aim`/`leave` as `innermost` changes on `onPage`, calls `leave` on the aimed receiver on cancel — including the cancel `createDrags()` issues when its owner is disposed (D7) — and on release calls `drop` only if `landings.standing(receiver)`. It takes an optional `held` flag so a touch carrier can start at threshold zero after its long press, as `dragging.ts:406` does.

Unit: `carry.test.ts` — register/release; `lift` skips receivers whose `lift` answers null; `innermost` prefers the smaller box and answers null outside all; a receiver released mid-carry is not `standing`; two apps have two tables.

e2e: none yet.

### 6.2 A composer takes a row

Chat's `Conversation` (`Fold.tsx:38-51`) registers one receiver around the `DropTarget` element it already renders, for the component's lifetime. Its `lift` measures that element for every kind (nodes, text, file) and answers null when the conversation draws `Unopened` (no box, no composer). `aim` sets a `carrying` signal the `DropTarget` reads to light its overlay with the sentence for the kind: *drop to ask about it* / *…about them* for nodes, *drop to quote it* for text, *drop to name it* for a file; `leave` clears it. `DropTarget` keeps its HTML5 `Files` path untouched and gains a `carrying` prop; the overlay keeps `data-testid="chat-drop"` and gains `data-carrying`.

`drop` for nodes: `ui.armed.armNode(id)` for each id in order — the `ConversationUI` this composer already holds (`Composer.tsx:161`), so the existing focus effect (`Composer.tsx:402-409`) puts the caret in the box. Nothing is sent. Text and file drops come in 6.4 and 6.5; until then `lift` answers null for those kinds.

Outlines' row drag consults the table: `createDragging` takes `landings` (handed down from `browser.tsx` where `createFields`/`createAir` are made, `packages/plugins/outlines/src/browser.tsx:111-113`, held in the activation). In `lift()` (`dragging.ts:393`) it snapshots `landings.lift({ kind: "nodes", ids, file })` beside `measure()`. In `onPage`, `innermost` is asked first: inside a receiver the aim becomes `{ kind: "receiver", lifted }` — the drop line and refusal are not drawn, the receiver's `aim` is called, and leaving it calls `leave`. `onEnd` with a receiver aim calls `drop` and moves nothing; a refusal string goes to `selection.say` like a refused move. Escape calls `leave` on the aimed receiver. The `Aim` union in `drag/aim.ts:73` gains that arm; `Aiming.tsx` draws nothing for it.

Held-service holder: chat gets `holdLandings`/`landings()` in a module beside `browser/vault.ts:13`, acquired in the activation with `needs: [Landings]` on the main component (it never waits: the host supplies it). Outlines the same.

Unit: `dragging.test.ts` (or the nearest existing test of `aimAt`) — a point inside a receiver box yields the receiver aim over a page gap at the same point; outside, aiming is unchanged. `DropTarget` sentence table.

e2e, new `drop_rows_into_a_conversation.feature`:
- a bullet carried onto an open fold's panel lights *drop to ask about it*; released, the row is a chip and the row has not moved; sending carries the node (`the composer is armed with …` steps from `node_context_steps.ts:39-65`, and the fake agent's `name <id>` reply as `node_context.feature:29` does);
- two picked rows dropped arm both in pick order; the chip's × takes one off;
- a row from `lanes.olai` in pane 1 dropped on the zoomed `orchestrator` page in pane 2 arms it — no cross-file refusal, no `drop-refused`;
- the panel stops lighting when the row leaves it and the drop line returns over the outline;
- Escape over the panel arms nothing and moves nothing; ⌘Z has nothing to undo;
- the fold is closed from another tab (or the plugin runtime rebuilt, as `drag_live_recovery.feature` does) while the row is held over it: release arms nothing, moves nothing, and a fresh drag works;
- on a phone (`on_a_phone.feature` style, `holdDown`/`dragFinger`): a held row let go over the fold arms it.

### 6.3 Ask agent goes

Delete the `ask-agent` entry from `rowVerbs` (`verbs.tsx:35-41`); keep `target`, `show`, `NO_AGENT` for `createAskCommand`. Update the comments that use it as the worked example in `packages/plugin-api/src/browser.ts:157` and `:192` (name *Start an agent session* instead). `menu_panel.feature:79` already asserts the core menu has no such entry; add to the new feature one scenario that the row menu of a child under an agent offers *Start an agent session* and not *Ask agent*.

Re-point scenarios that reached a conversation through the menu: `node_agent_folds.feature:60` becomes the drop (moved to 6.2's file), `:106` keeps only its palette half, `:167` becomes "the palette finds the ancestor while search is unavailable"; `node_context.feature:29`, `:48`, `:91`, `:215` arm by drop; `chat_at_nodes.feature`, `the_agent.feature:2151`, `codex_steering.feature:86` wherever they say *Ask agent*. Unit: `verbs` test that `rowVerbs` yields only start entries.

Docs in 6.7.

### 6.4 A transcript row carries its words

A grip per row that has words: `Row.tsx` draws it at the left edge for `agent` (not while `streaming`), `user` and `tool` entries; `Diff.tsx` and `OutlineDiff.tsx` draw their own inside the box head. Hidden until hover/focus (`HOVER_REVEAL`, `packages/ui-primitives/src/touch.ts`), primary button only, `data-grip` and `data-testid="chat-grip"`; a press that does not travel does nothing. On pointer-down it calls `carry(event, { kind: "text", text }, landings(), { threshold: TRAVEL_PX })` with the text D5 defines, computed at lift from the entry (`chat.entry(key)`), not from the DOM. A pure `textOf(entry)` / `textOfDiff(path, diff)` in `browser/chat/carried.ts` with unit tests for each kind.

Touch (D8): the grip is not the handle on a phone — there is no hover — so the row itself takes `longPressOn` (`packages/web/src/client/longPress.ts:124`) the way `dragging.ts:454` wires the bullet: after `LONG_PRESS_MS` the row lifts and `carry()` starts with `held: true` (threshold zero); a `touchmove` before the deadline is scrolling and claims nothing. The row's own `•••`-style long press does not exist on transcript rows, so nothing is displaced; a long press on a link or a pressable id inside the row stays the browser's.

The composer receiver's `lift` now takes `text`; `drop` writes the quote at the caret: a pure `quoted(text)` (`> ` per line, one blank line after) and `written(draft(), { from: caret() }, quoted, caret())` (`packages/markdown-ui/src/insert.ts:20`) through the composer's `rewrite` (`Composer.tsx:523`) — expose `rewrite` on the `ConversationUI` or lift it into the receiver; either way the DOM write stays the one `rewrite` makes, for the reason its comment gives. Unit: `quoted` on one line, several, trailing newline, empty lines; insertion inside a sentence keeps the caret after the quote.

e2e, new `drop_transcript_rows.feature`:
- an agent's answer carried from fold A to fold B's panel lights *drop to quote it*; released, B's box holds `> …` at the caret with the sentence before and after it intact; sending delivers the quote (fake agent echo);
- a tool row quotes title and reply; a diff box quotes its `+`/`-` lines; your own message quotes;
- a streaming answer (`stream slow`) shows no grip until it ends;
- dropped on its own conversation's panel it quotes there;
- Escape mid-carry writes nothing; the fold closing under the held row writes nothing and a fresh carry works; the plugin runtime rebuilt under a held row leaves no panel lit (D7);
- pressing a grip without travelling changes nothing (no draft change, no selection);
- on a phone: a held agent answer lifts and, let go over another fold's panel, quotes there; a flick that starts on a row scrolls the page and quotes nothing.

### 6.5 A sidebar file row carries its path

Files' row: pointer-down starts `carry(event, { kind: "file", path }, …)` with the pin's element pattern (5.3). Folders are not handles. A travelled drag swallows the click that follows so release over a composer does not navigate; a press without travel still opens the file. Files' browser component declares `needs: [Landings]` and holds it itself.

Touch (D8): a held file row lifts after `LONG_PRESS_MS` through the same `longPressOn`; a finger that moves first scrolls the tree. On a phone the lift closes the drawer per 5.6, so the carry continues over the page.

The composer receiver's `lift` takes `file`; `drop` writes `@${path} ` at the caret through the same `written`/`rewrite`, reusing `inserted(path)` from `browser/chat/completion.ts:159` so the two doors cannot drift. No node is armed and `taken` is untouched — a file is a word (`docs/chat.md` *What it writes is a word, not an attachment*).

e2e, new `drop_files_into_a_conversation.feature`:
- a file row dropped on a panel writes its path at the caret inside a sentence; sending reaches the agent with the path as text (the fake agent's echo);
- press without travel opens the file; a folder row is not a handle;
- a file carried over an outline page lights nothing and release does nothing;
- Escape writes nothing;
- on a phone: a held file row lifts, the drawer closes, and let go over a fold's panel the path is written; a flick that starts on a row scrolls the tree and the drawer stays.

### 6.6 A transcript row lands in an outline

Outlines' `Editable` registers a receiver per editable page beside `useFields().join`. `lift` for `text` measures the page's box and its rows once (the per-page half of `measure()` at `dragging.ts:272-310`, extracted so both use it; `placeable` with nothing held). `aim` runs `planDrop(rows, x, y)` (`drag/plan.ts:237`) and shows the landing through the page's `Aiming` — the `aim` signal `Aiming` reads becomes `dragging.aim() ?? foreign.aim()`. `leave` clears it. `drop` sends one `add` edit through `applying(edit, undo.record)`, where the landing's `{ parent, after }` maps to the `Anchor` `add` takes (`packages/surface/src/edit.ts:208-247`): `after` names `{ kind: "after", id }`, a null `after` under a parent is `{ kind: "under", id }`, and a null `after` at top level is `{ kind: "first", file }` — the same mapping `landingFor` (`packages/edit-intents/src/index.ts:344`) already reads. The note rides the same op: the surface `add` edit carries only `title` today (`edit.ts:240-247`) while the ops-level `AddRequest` already takes `desc` (`packages/format/src/writing.ts:89-91`, `:354`), so widen the edit with an optional `desc` and pass it through `addRequest` (`index.ts:376-383`). That is a static contract change in `@olai/surface` and `@olai/edit-intents`, not a new op, and it keeps the drop one write and one undo entry; the `desc` verb (`edit.ts:405`) is not used because the new node's id is unknown until the add lands. A refusal is spoken on the selection bar as a refused move is. Every kind but `text` gets `lift: null`, so a row move never sees its own page as a receiver.

Unit: `landing.test.ts` — title/note split (one line, several, leading blank lines, trailing whitespace); the landing-to-anchor mapping for first-child, after-sibling and top-level.

e2e, added to `drop_transcript_rows.feature`:
- an agent's one-line answer dropped between two rows becomes a node there, at the depth the line promised (`the drop line would put it …` steps from `dragdrop_steps.ts:112-132`), and ⌘Z removes it;
- a several-line answer becomes a node with the first line as title and the rest as note;
- dropped into the other pane's outline; dropped into a zoomed page's subtree;
- a refused add (the outline is deleted while the row is held — `drag_live_recovery.feature`'s shape) says why and creates nothing;
- carried over a document page nothing lights.

### 6.7 Docs

Same PR. Rewrite, do not append:

- `docs/chat.md`: `## Asking about one node` becomes *Handing things to a conversation* — rows by drag (and across files and panes), transcript rows as quotes, files as paths, what a row carries (D5), the plain composer and touch limits; the `>` paragraph stays; `:250` and `:252` say *a dropped row* where they say *Ask agent*; `## Attachments` keeps its OS-file paragraph and gains one sentence that the same panel takes rows, transcript rows and files.
- `docs/editing.md` *Dragging a row*: a row let go over a conversation is armed there and does not move; a transcript row let go between rows becomes a node; the cross-file refusal is about outlines only.
- `docs/plugins/chat.md` (symlink to `packages/plugins/chat/docs.md`): seat table row `outline.row.action` → *Start an agent session*; a row for `landings` (host-supplied, what chat registers, what it carries); the optional-dependency paragraph at `:87-92` loses *Ask agent's server lookup* and keeps the `>` sentence.
- `docs/plugins/outlines.md`: the page receiver and the row carry consulting the table. `docs/plugins/files.md`: file rows carry their path.
- `docs/architecture/cordis.md` *Choose the owner before the helper* table: one row for `Landings` (host-supplied, per app, registrations component-owned). `docs/architecture/slot-ownership.md` needs no row — it is not a location.
- `docs/architecture/overview.md:237` is stale already (cross-pane row drags landed in `drag_across_panes.feature`); fix it in passing and name the table.
- `docs/running.md:258` drops *Ask agent* from the conversation's list.
- `docs/architecture/e2e-coverage.md`: Outline structure, Chat drafts and context, Node agents and Panes rows name the three new feature files and the re-pointed scenarios; the *Ask agent* mention at `:66` goes.
- `website/`: nothing there pictures the row menu or says *Ask agent* (grep found none); leave it.

## 7. Cordis checklist for the reviewer

- [ ] `Landings` is supplied by `openApp`, once per app; no plugin offers it, no module-level table exists, two apps in one page have two tables.
- [ ] Every receiver registration is released by its component's cleanup and again, harmlessly, by its activation's holder release; a release never removes a later registration (token per hold, as `heldService` does).
- [ ] A carrier holds only a snapshot and asks `standing` at release; a receiver gone mid-carry receives nothing and the carry ends quietly.
- [ ] Chat names `Landings` on its main component and nothing of outlines' or files'; outlines and files name `Landings` and nothing of chat's. No plugin imports another's drag or composer implementation; `carried.ts`, `quoted`, `textOf`, `planDrop` stay in their owners.
- [ ] `plugin-api`'s `carry.ts` names no feature: `Carried` is `{ kind }`; each payload's shape and guard sit in its carrier's `/carry` door, and a receiver imports only those guards (types and pure functions, D2a).
- [ ] Disposing a carrier's component mid-carry cancels the gesture and calls `leave()` on the aimed receiver; no overlay or drop line survives its carrier (D7).
- [ ] The outline row move is unchanged when no receiver is under the pointer: same aiming, same refusal, same `place` ops, same touch path.
- [ ] Optional availability: with chat off, rows drag exactly as before and transcript grips do not exist; with outlines off, a transcript row over a document lights nothing; with files off, nothing changes for chat.
- [ ] The `>` command, `agentAbove` and the two refusals are untouched; `rowVerbs` yields only start entries.
- [ ] `data-handle` stays outlines' monopoly in `claims.test.ts`; new attributes are added to that table.
- [ ] Reconnection: a carry does not touch the wire until release; a drop during the reconnect gap is refused in words by the existing write path, not lost.

## 8. Decisions, approved

Approved by the human on 2026-09-12:

1. **Pointer carries, not HTML5** (D1), so OS files stay the only HTML5 drop.
2. **The table lives on the host** (D2) as `Landings` in `@olai/plugin-api`, supplied by `openApp` like `Edits`; **payload shapes live in each carrier's contract door** (D2a).
3. **What a transcript row carries** (D5): the row as drawn is the grain; a tool row carries title plus reply; a diff carries its lines; no grip on streaming rows.
4. **Quote shape**: `> ` per line, a blank line after, no attribution line.
5. **Into an outline**: first line title, rest note (D6).
6. **Touch carries for transcript and file rows are in** (D8). **The plain node's dashed composer as a landing is out** (section 2).

## 9. Out of scope, noted for the next PR

- The plain node's dashed composer taking text and files (arming needs a conversation to arm in; a quote or a path could be kept in its draft through `keepMessage`).
- A file row dropped into an outline as a node with a link, if it turns out to be wanted.
