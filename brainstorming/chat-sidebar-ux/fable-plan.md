# Node agents live in the outline

Implementation plan for one PR on branch `chat-sidebar-ux`. Written for an agent that has not seen the discussion; every claim below about the current code was checked against this tree at `83bf8ba19`. Where a fact still has to be found out, the item says so and says what to do in either case.

The design being implemented is the canvas at https://claude.ai/code/artifact/c3a963a7-eb67-462b-952f-3f302e2d1927 (five screens, notes beside each). Read it before the code.

## 1. What changes for a person

Today a node agent has four faces (sidebar roster row, `chat-agent-session` chip, door card, right-hand panel header) with four different information sets and three different click behaviours, and the conversation lives in a fixed right dock. After this PR:

1. **The row is the agent.** An agent-carrying row shows its standing in the aside slot beside `3/7`: engine mark, dot, word (`roster.ts`'s `LOOK`), a clock while working, an age when asleep. No session chip is drawn in outline view.
2. **Pressing the standing unfolds the conversation in place**, under the row and above its children: agent line (mark, model, usage, working cue, `open the page ›`), the transcript, what is running, the composer. Several rows in one file can be unfolded at once. Pressing again folds it.
3. **Zooming into the node is the full page**: breadcrumb and `h1` as today, the agent line under the title, the property in the zoomed drawer (folded), the subtree, then the conversation and composer in the same pane, unbounded.
4. **Starting an agent is one press**: hovering any row without an agent shows a dashed `start an agent` pill in the same aside slot. The row menu entry stays (keyboard, phone long-press, engine choice). A zoomed plain node also carries the composer; sending starts the agent.
5. **The sidebar's Agents roster is gone.** Two regions replace it: *Needs you* (standings `needs-you` and `gone` only) and *Recent* (every node agent regardless of standing, by last activity, capped at 10). An asleep agent is listed in *Recent* like any other; what is never drawn in the sidebar is the word `asleep` or any standing at all, only the age. Agents beyond the cap are reached through the palette.
6. **The right dock, the `>_ agent` header toggle, the phone chat sheet, `+ new`, and `sessions (n)` are retired.** Fresh start is one control on the agent line. Past sessions are reachable from the fold line above the transcript.
7. **A conversation no node claims is filed into the Inbox as a node, automatically.** Whenever the server learns of an unclaimed conversation (at boot, and each time the stored listing changes), it mints one node under a `Chats` node in the Inbox file carrying the session property, so it is an asleep node agent from that frame. *Move to…* then gives it a real home; the property travels with the node. *Unassigned*, *assign to node…* and the in-panel chat list are retired. There is no unclaimed state a person can see.
8. **`+ new` stays, as a command:** it mints a node under `Chats` in the Inbox, starts its agent, and unfolds it. Every chat is a node from its first message.

## 2. Non-goals

- No change to the ACP wire, engines, teaching preamble, scheduler capacity or idle policy, `heard/` storage, or the vault property format.
- No new persistence outside the vault. Fold state is tab-local UI state. The filer writes nodes, nothing else.
- No redesign of the transcript rows, tool frames, diffs, lanes, strips, composer, `@` completion, attachments, or questions. They move; they do not change.
- Sidebar Agenda, Inbox, month, pins, and files are untouched.

## 3. Ownership after the change (Cordis)

Keep these boundaries exactly. Every value that crosses a package boundary at runtime goes through a declared service, slot, or wire cell; nothing is imported as a live value.

| Thing | Owner | How others reach it |
|---|---|---|
| Slot names `outline.row.aside`, `outline.row.fold`, `outline.page.head`, `outline.page.foot`, and the kind-keyed `outline.row.placement` | `olai-plugin-outlines` (`src/slots.ts`) | `slots.register(...)` from chat |
| Which folds are open in this tab | `olai-plugin-chat` browser, one module beside `browser/agents/showing.ts` | not shared; the fold face reads it |
| A conversation's transcript, state, saying | server, one subscription per `Conversing` per tab | keyed wire cells/collections (§5.2) |
| Which conversations this tab is reading, and their liveness hold | chat server (`scoped.ts` scheduler) | acquired by subscribing, released by unsubscribing |
| Roster (`cells.agents`) and `useAgents()` | chat browser `AgentsProvider`, once per activation | context, as today |
| The Inbox file's path | `olai-plugin-capture` | a declared service chat names optionally (§5.9); with no capture plugin the filing gesture is absent |
| Minting nodes for unclaimed chats | chat server's filer, through `Ops` (already named in `server.ts:196`), owned by the chat server scope | runs on its own; `conversation.newChat` is the one procedure that mints on request |
| Panel open/width/snap (`layout.shell`) | layout | **no longer read by chat** after this PR; leave the service in place |
| `node.changed`, `said.at` | ops layer / `heard.ts`, as today | already on the outlines collection and the roster cell |

Removed faces release what they held: the `app.panel` registration and its `trackCamera` acquire/release, the `app.header` Toggle, the mobile sheet and `Minimized` strip, `holdShell` and `browser/shell.ts` readers. Do not leave a service named-but-unread (see the header of `packages/plugins/chat/src/browser.tsx:38-56` for why that hangs a fiber).

## 4. Find out first

Do these before writing code. Each has a fork.

**4.1 Keyed collections on the wire.** Read `packages/plugins/chat/src/wire.ts` and the wire package it builds on (follow `chatWire()` in `packages/plugins/chat/src/browser/wire.ts`). Today `cells.state`, `collections.transcript` and `collections.saying` are single and describe the server's one foreground conversation (`browser/chat/state.ts:233-245`). This PR needs one subscription per `Conversing = {agent, session}`.
- If the wire package already supports keyed cells/collections (a `.use(key)` form), use it.
- If not, add the primitive to the wire package (that package owns it), with subscribe/unsubscribe as acquire/release, and a unit test that two keys are two subscriptions and releasing one leaves the other. Do not fake it with one collection carrying a conversation tag on every row.

**4.2 Where the server's "foreground" is used.** `grep -rn "selected\|foreground\|loadSession" packages/plugins/chat/src/server.ts packages/plugins/chat/src/chat.ts packages/plugins/chat/src/scoped.ts`. The notion of one selected conversation per tab goes away; "selected conversations prevent timed eviction" (`docs/chat.md:600`) becomes "conversations any tab is reading prevent timed eviction". Keep pending questions and active background tasks as eviction blockers exactly as they are.

**4.3 The property key spelling.** `packages/plugins/chat/src/kinds.ts` composes the key (`SESSION_TYPE`, from `binding.ts`). Docs disagree with each other (`docs/chat.md` says `chat-agent-session`, `docs/plugins/chat.md` and `node_agents.feature` say `agent-session`). Use whatever `kinds.ts` declares in every line of prose and every fixture you touch, and fix the disagreeing doc in passing.

## 5. Work items, in commit order

Commit after each numbered item; run `just typecheck-fast-remote` after 5.1 to 5.4 and `just e2e-fast-remote` from 5.5 on. Each item names its e2e coverage; write the scenario in the same commit as the behaviour.

### 5.1 Three slots in outlines

`packages/plugins/outlines/src/slots.ts`: declare

- `outline.row.aside`: face `(props: {readonly node: string}) => JSX.Element`, `keyedBy: "nothing"`. Drawn by `NodeLine.tsx` after `ProgressBadge`/`Aside` and before `DateBadge`, on every row including the zoomed subject's title line in `NodePage.tsx`. `Tree.tsx:708` and `DatedRow.tsx:147` pass `aside` as a prop today; keep that prop and draw the slot beside it. Add a `PluginAsides` component beside `Doors.tsx`.
- `outline.row.fold`: same face, drawn by `NodeBody.tsx` on an ordinary row between the `PluginDoors` site (`:200`) and the note line. This is where the unfolded conversation lands.
- `outline.page.head`: same face, drawn by `NodePage.tsx` under the title line and above the zoomed subject's `NodeBody` (the drawer at `NodePage.tsx:181` belongs to that `NodeBody`, so this is the only mount point between title and drawer). The zoomed agent line lands here.
- `outline.page.foot`: same face, drawn by `NodePage.tsx` after the children `Tree`. The zoomed conversation and composer land here.
- `outline.row.placement`: face `{readonly inRows: boolean}`, `keyedBy: "kind"` like `outline.row.chip`. A property kind's one word about where its ordinary chip is drawn. Read by the outlines row when it builds `customEntries` (row view): a kind whose placement says `inRows: false` is left out of the run there, and nothing else changes; `drawerEntries` (the zoomed page) ignores placement and draws every property as today. No filler ever replaces or redraws the ordinary chip; outlines keeps the only renderer. Absent contribution means `inRows: true`, so every existing kind is unaffected.

Register the four in `docs/plugins/chat.md`'s seat table (`## Where it hangs in the tab`) with "who declares it" and "what chat brings". Unit test the slot table the way the existing names are tested.

e2e: none yet; the faces arrive in 5.3 and 5.5.

### 5.2 Conversation-keyed reading

Per 4.1. Server (`packages/plugins/chat/src/server.ts`, `server/*.ts`): expose `state`, `transcript`, `saying` keyed by `Conversing`. Subscribing acquires the node scope if the conversation belongs to a node agent (same path `loadSession` takes today); unsubscribing releases the reading, and the scheduler's idle reaper treats "read by some tab" the way it treats "selected" today. Sending, cancel, interrupt, retry, try-again already carry conversation identity per `docs/chat.md:163-167`; verify each procedure's input names the `Conversing` and drop any that relied on "the selected one".

Browser (`browser/chat/state.ts`): `createChat(conv: Conversing): Chat`. The tab-scoped `refused` signal (`state.ts:243`) becomes per-`Chat`. `createChatState()` with no argument is removed; find every caller (`browser.tsx:123-127`, `verbs.tsx`, `answered.tsx`) and pass a conversation or, where the caller only needs the roster of engines, split that out (`chat().roster` is engine discovery, not a conversation; give it its own cell or leave it on a keyless `engines` cell).

`loadSession` becomes a read-side no-op for node agents (subscribing is opening). Nothing else needs it after 5.9; delete it and its docs.

Unit tests: `state.test.ts` for two `Chat`s over two conversations; scheduler test in `scoped.test.ts` that a read conversation is not reaped and an unread idle one is.

e2e (`node_agent_live_history.feature`, `node_agent_idle_lifecycle.feature`): update the idle scenario so the thing that holds a scope live is an unfolded row, not a selected panel.

### 5.3 The standing in the aside

New `browser/agents/Standing.tsx`, registered on `outline.row.aside`. For a row in `useAgents().at(node)`:

- mark (`AgentMark` for the engine), dot and word from `LOOK`, `· <clock>` while `working`/`waking` (reuse the elapsed readout from `chat/lanes.ts`/`ElapsedProvider`), `· <ago>` from `said.at` when `asleep` (reuse `agoOf`/`createNow` as `Door.tsx` does). `needs-you` draws the word in `text-doing`.
- pressable when `session !== null`: toggles the fold (5.5). An unbound agent's standing is not pressable, as the door is not today.
- `data-testid` and `data-standing` on the element; extend `packages/plugins/chat/src/testids.ts` (one object literal, no spread; `packages/bundle/src/testids.test.ts` catches duplicate values).

For a row with no agent: a `start an agent` pill, hidden until row hover/focus (`HOVER_REVEAL` from `packages/ui-primitives/src/touch.ts`), dashed `QUIET_PILL` shape with the mark. Pressing runs the same code path as `verbs.tsx`'s start verb: with one engine, `conversation.startAgentSession({node, agent})` through `runAsync`; with several, a small menu of engine names (reuse the row menu's `Dropdown`/`Panel` classes). On success the row unfolds (5.5). Refusal text lands on the row as a `SaidLine`, the way the roster section draws `agent-refused` today. A row already talking gets no pill (the existing verb rule at `verbs.tsx:99-120`).

Delete `Door.tsx` and the `outline.row.door` registration; the `needs-you` question moves into the fold (5.5). Keep the slot declared in outlines if any other plugin fills it; grep first.

e2e (`node_agents.feature`): rewrite "stands on both its faces" scenarios so the two faces are the sidebar *Needs you* row and the aside standing; add: hovering a plain row shows the pill; pressing it writes the property and unfolds; a bound row shows no pill; two engines installed gives a menu (`@codex` tag).

### 5.4 The property chip is not drawn in outline view

Register `outline.row.placement` keyed by the session kind with `{inRows: false}` (5.1). That is the whole contribution: the outlines row omits the kind from `customEntries`, so a bound row draws no session chip in outline view, while the zoomed page's `drawerEntries` draws the ordinary chip exactly as today: a plain-text value, whole (an `<engine>:<uuid>` binding is 43 characters, under `PropsDrawer.tsx`'s 56-character fold, so it draws on one line; a longer value folds into the existing disclosure, and that is fine). No clamping change for this kind. Do not register `outline.row.chip` for this kind, and do not add a replacement mode, a placement context, or an exported chip renderer to the chip slot; the ordinary chip stays outlines' own. Editing the property by hand stays possible on the zoomed page; document that in `docs/chat.md` where "Re-pointing a bound node by hand is still an edit to the property" is said, and say that the chip is drawn on the page only.

e2e: a bound row draws no property chip in the outline while its other custom properties still draw; the zoomed page draws it as an ordinary chip with the whole value; a kind with no placement contribution draws in rows as before; `node_agent_mutations.feature` keeps proving that editing the chip on the page re-points the agent. Unit: outlines' entry builders honour `inRows: false` in `customEntries` and ignore it in `drawerEntries`.

### 5.5 The fold

New `browser/agents/Fold.tsx`, registered on `outline.row.fold`. Draws only when this tab has the node unfolded. Fold state: a module beside `showing.ts` holding `Set<node id>` in a signal, with `unfold(node)`, `fold(node)`, `unfolded(node)`. Tab-local, in memory, not persisted.

Contents, top to bottom, all existing components passed a `Chat` built by `createChat(conv)` for that node's `Conversing`:

1. Agent line: `AgentMark` + engine name, `Model` trigger, usage, working/waiting cue (the second line of today's `Header.tsx:162-218` extracted into its own component, `AgentLine.tsx`, used here and in 5.6); at the right, `open the page ›` (a `Link` to `atNode`), and `fresh start` (5.7).
2. `Plan`, `Roster`, `Watching`, `Wake` strips, unchanged.
3. `Transcript` inside a bounded pane (`max-h-[24rem] olai-scroll`), opening on its newest line as today.
4. `Busy`, then `Composer` with `createHolding`, inside one `ElapsedProvider`.
5. A `needs-you` turn draws its question form here, exactly as the panel draws it today (`chat/asked` and the question components); no separate door card.

Subscription ownership: the fold mounts `createChat(conv)` inside a Solid owner that is disposed when the fold closes or the row leaves the page, and disposal releases the wire subscription. Two folds are two owners.

`Ask agent` (`verbs.tsx`): on a row that is an agent or under one, arm the node into the nearest ancestor agent's composer and unfold that agent; on a row with no agent above it, return the refusal sentence "no agent above this row — start one". `chat/attention/reveal.ts` (alerts): a notification click still carries only `{kind: "ask"}`; the alerts contract from #578 keeps click payloads free of identity on purpose, and this PR does not touch that union or its decoder. On a click, reveal navigates to and unfolds the **first agent in the roster's needs-you order**, the same order the sidebar's *Needs you* region draws (5.8), so what the click opens is the row at the top of that region. With several agents waiting, the region is the disambiguation, one press away. With none waiting by the time the click lands (the question was answered elsewhere), reveal does nothing beyond focusing the sidebar's *Needs you* region if it is drawn, and otherwise nothing.

Delete: `app.panel` registration, `Panel.tsx`'s `DesktopDock`, mobile sheet, `Minimized.tsx`, `Toggle`, `browser/shell.ts` and `holdShell`, `showing.ts`'s panel-only pieces, `Choose.tsx` (there is no free chat to choose an engine for), `chat/open.ts`. Remove chat's `lg:pr-[var(--width-panel)]` reservation in `packages/plugins/layout/src/Frame.tsx` only if no other `app.panel` filler exists; otherwise leave layout alone.

e2e: new `node_agent_folds.feature`: unfold draws the transcript and composer under the row; sending from a fold writes the vault through that node's session; two rows unfolded at once each answer their own send; folding releases (the idle reaper can take it; use `@node-idle-fast`); a needs-you question is answered from the fold; a notification click lands on and unfolds the first needs-you agent, and with two waiting it is the one the *Needs you* region lists first (the existing alerts scenarios that asserted the panel opened are re-pointed here; the click payload stays `{kind: "ask"}`); a click with nothing waiting opens no fold; `Ask agent` on a child row arms it in the ancestor's composer; reload folds everything. Move every scenario in `chat_*.feature` that asserted on the dock to the fold or the zoomed page; delete scenarios about the toggle, the sheet, the minimized strip, and `+ new`.

### 5.6 The zoomed page

Register `outline.page.head` with `AgentLine` (mark, model, usage, cue, `fresh start`; no `open the page ›`, since this is the page) and `outline.page.foot` with the `Fold` component in an `unbounded` variant (no `max-h`, the pane's own scroll) and without its own agent line. `NodePage.tsx` draws the `outline.row.aside` slot on the title line so the standing sits beside `3/7`. Order on the page is then: breadcrumb, title line with standing, agent line (head slot), drawer with the property chip, note, children `Tree`, conversation and composer (foot slot). Both slot faces build their `Chat` from the same `createChat(conv)` for the node, under one owner per page, so the page holds one subscription, not two.

A zoomed plain node draws the foot too: a dashed composer with placeholder `ask about <title>…`, an engine picker (default: the machine's first engine, as `verbs.tsx` orders them), and the notice `sending starts this node's agent · memory: this subtree (<memoryOf>)`. Sending runs `startAgentSession` and then the queued message goes into the opened conversation (the "message typed while a conversation is opening waits for it" rule at `docs/chat.md:181` already covers the gap; reuse it).

e2e: `node_agent_page.feature`: zoom draws memory then conversation; `open the page ›` from a fold lands there with the fold's draft intact (drafts are per conversation, `docs/chat.md:94`); typing under a plain zoomed node starts an agent and delivers the message; the property chip is on the page.

### 5.7 Fresh start and history

`NodeSessions.tsx` is retired as a menu. Its two halves move:

- `fresh start` is a control on the `AgentLine`, with the existing sub-sentence as its tooltip and the existing double-press guard. Same server verb, same ordering (open, then write).
- The fold line above the transcript (`since <date> · N messages above the fresh start ↑`) is pressable and lists that agent's past sessions using `Conversation.tsx` rows. Picking one swaps the fold's `Chat` to that conversation (`createChat` over the past `Conversing`, the previous one disposed, the fold's owner unchanged), with the fold line then reading `past session · <minute> · current session ↩`. A past session is **writable**, exactly as today: it opens in its own node-scoped process, its composer sends, and its tool writes reach the vault, including after a server restart (`docs/chat.md:711-715`, `node_agent_fresh_sessions.feature:93`, `node_agent_history_file_recovery.feature`). What it never does is rewrite the node's property: the binding still names the current session, the standing in the aside still reports the current session, and `current session ↩` swaps the fold back. Nothing here is read-only; the earlier draft's "read-only" was wrong. Past sessions come from `pastOf` in `lineage.ts`, unchanged; the fold line's list and the `sessions (n)` count it replaces are the same `pastOf` answer.

e2e: `node_agent_fresh_sessions.feature`, `node_agent_fresh_refusal.feature`, `node_agent_pending_fresh.feature`, `node_agent_removed_fresh.feature`, `node_agent_live_history.feature`, `node_agent_history_file_recovery.feature`, `node_agent_phone_history.feature`: keep every rule, including "a reopened past session writes the vault, after a restart too, with the binding unchanged" and "the current session keeps working while history is open, and the sidebar reports the current session's standing"; re-point the steps at the agent line and the fold line (`sessions (n)` becomes the fold line's list; `current session` becomes `current session ↩` on the fold line). Delete only the phone-sheet-specific steps.

### 5.8 Sidebar

`browser/agents/Agents.tsx` becomes two `sidebar.section` registrations (both under `AgentsProvider`):

- **Needs you** (`said: "Needs you"`): rows from `useAgents().rows()` with standing `needs-you` or `gone`, in that order, then by `said.at` desc. Row shape: `ENTRY_SHAPE` + `DOT` + title + `CHIP_QUIET` count of waiting questions, or the word `not running` in the `ago` slot. Press: navigate to the row (`atElement`, never `atNode`) and unfold it. Section draws nothing when empty.
- **Recent** (`said: "Recent"`): every node agent, whatever its standing (asleep, idle, working, needs-you, gone, unbound alike), those with a `said.at` ordered by `said.at` desc, then those without one by `node.changed` desc (available on the outlines collection; read it through the roster row's node, or extend `NodeAgentRow` with `changed` on the server side so the browser does one subscription, not two). Cap at 10 after sorting. Row: `ENTRY_SHAPE` + `AgentMark` + title + `ago` in mono; no dot and no standing word in this section (a row that needs a person is already in *Needs you* above, and may appear in both). Current row (`aria-current="page"` when the route is that node's file and the row is unfolded or zoomed) gets the existing `aria-[current=page]` wash. A `new chat` row heads the section (`ENTRY_SHAPE`, `text-paper/65` like the file doors), running `conversation.newChat` (5.9). Draws only the `new chat` row when there are no agents. The old *Unassigned* row is gone (5.9).

Add an `app.command` "Agents" (palette) listing every node agent by title with standing, in the same activity order, so the agents beyond the cap are one keystroke away; picking one navigates and unfolds.

Delete the old `Agents` section and `focus.ts`'s panel half (keep `rowOf`).

e2e: rewrite `node_agents.feature`'s roster scenarios: the roster is the query still, but drawn as *Needs you* and *Recent*; an asleep agent is in *Recent* with its age and no standing word, and not in *Needs you*; a needs-you agent is in both; ordering by activity; the cap, with the eleventh agent absent from *Recent* and present in the palette; the palette command; `An unbound agent's sidebar press navigates` stays.

### 5.9 Filing unclaimed conversations into the Inbox, automatically

The unclaimed set is what `lineage.ts`'s `unassignedIn` computes today (the stored listing minus what the roster claims, `/clear` chains included). A server-side **filer** turns every member of that set into a node, with no press.

**Where the Inbox is.** `packages/plugins/capture/src/Inbox.tsx` gets `props.file`; find how capture learns that path and whether the server half of capture knows it too. If capture exposes no service for it, capture declares one, server side (`serviceTag<{file: Effect<string>}>("capture.inbox")` or the shape its existing contracts use), and chat's server names it **optionally**, as a sub-component that does nothing when the service is absent: with no capture plugin there is no filer, unclaimed conversations stay unclaimed and invisible, and the log says so once at boot. Do not make chat's whole server scope depend on capture.

**When it runs.** Owned by the chat server scope, started with it, stopped with it. Never on a clock. Two kinds of run:

- A **full run**, asking every installed engine the way `conversation.sessions` does (the running agents live; the others started, asked and stopped, one at a time, with the same few-second reuse): once after the engines have been probed at boot, and on every `sessionsRevision` bump (a fresh start, a `+ new`).
- A **narrow run**, after every settled node-agent turn, asking **only the engine whose turn settled**: it is already running, so the question costs one live request and starts no subprocess. This is what catches a conversation created outside olai (`claude --resume` in a terminal) once anything in that engine settles, since the roster names every filed chat and a "conversation the roster does not name" can no longer be the trigger.

An agent that could not be asked is logged with its reason and retried on the next run of either kind; its conversations are neither filed nor forgotten. The existing rule that a settled turn in an already-listed conversation never triggers another listing (`answered.tsx:39-89`'s once-per-conversation probe, and the scenario that pins it) is replaced by: a settled turn asks the one engine that settled, never another, and never starts one.

**What it writes.** Through `Ops`, as a sequence of independent writes, because an Ops batch is all-or-nothing (`packages/ops/src/batch.test.ts:345`) and one refused row must not undo the others. First, in its own write: ensure a `Chats` node exists at the top level of the Inbox file (create if absent, reuse if present, matched by a reserved id such as `chats`, never by title). If that write is refused the run stops there and is retried next run. Then one write per unclaimed conversation, in listing order (newest first), each minting one node under `Chats`: title = the conversation's own title (the agent wrote it), note = `<N> messages · last <minute>` as the listing says it, property = `sessionValue(engine, session)`. `/clear` chains: the head of a chain gets the node; the chain behind it is claimed by lineage exactly as `assignSession` claims it today (`docs/chat.md:696-699`), so it never becomes a second node. A refused row is logged with the validator's reason, leaves every other row filed, and is retried next run; a row that succeeded is never written again because the roster check runs per row before its write. Every write is an ordinary op: it appears in the Commit panel and commits like any edit. On a directory with many stored chats the first run is many writes in a burst, so what lands in the first commit is whatever the git plugin's cadence gathers, one large commit or a few; say so in `docs/chat.md`.

**Idempotence and identity.** A node is minted for a conversation only if no node in the set already carries its session (the roster is the check, `agentsIn` over the derived set). Trashing a filed node does not re-mint it: trashed nodes are excluded from the roster, so keep a second check against the trash before minting, or the filer would resurrect what a person put away. Renaming or moving the node changes nothing for the filer.

**`+ new`.** `conversation.newChat({agent})`: mint a node under `Chats` (title `new conversation`, no note), then `startAgentSession` on it, in that order, then answer with the node id so the browser can navigate to the Inbox row and unfold it. Offered as an `app.command` (palette) and as a `new chat` row at the head of the Recent region (5.8). With several engines installed the command asks which, using the row menu's engine list, not `Choose.tsx`.

The first message sent to a filed node uses the existing *assigned* contract (`teaching.ts`, `docs/chat.md:665`): the agent is told it was moved here and must distil what it knows into the subtree now. A `+ new` node uses the ordinary contract, since it was opened for the node.

**Second machine.** Session ids are machine-local. Machine B's filer mints nodes for B's stored chats; A's filed nodes draw on B with the existing "pressing it is refused by that machine's own agent" behaviour (`docs/chat.md:645-651`). Nothing new to design; state it.

**What it retires.** `Unassigned.tsx`, `showing.ts`, `assignSession` on the wire and in `binding.ts` (moving the node is the reassignment; `Move to…` keeps the property because it moves the node), `browser/agents/lineage.ts`'s browser-side `unassigned` consumers, the `+ new` button in the header, and `newSession` if no caller remains. `conversation.sessions` stays for the filer and for 5.7's history.

**Nesting.** A filed node moved under another node agent's subtree is an agent inside an agent's memory. The roster query already allows that; state in `docs/chat.md` that the inner agent's subtree is part of the outer one's memory and neither is refused.

Unit tests (`server/filer.test.ts`): a run mints exactly the unclaimed heads, the `Chats` node in its own write first and then one write per row; a second run mints nothing; a trashed filed node is not re-minted; a reused `Chats` node is not duplicated; a refused `Chats` write files no rows and is retried; a refused row leaves every other row filed, is logged with its reason, and is retried alone next run; a run interrupted between rows leaves the rows already written and files the rest next run; an unreachable agent's chats are skipped and logged; no capture service means no filer and one log line.

e2e (`node_agents.feature`, replacing the Unassigned scenarios; stored-sessions tag): a directory with stored chats boots with a `Chats` node in the Inbox carrying one row per chat with the property; the filed row is an asleep node agent that unfolds and answers with its context intact; `Move to…` on it keeps the property and the roster follows; trashing it does not bring it back; `+ new` mints, starts, and unfolds; a `claude --resume` from a terminal (scripted, as the existing stored-sessions scenarios do it) appears in the Inbox after the next settled turn of any agent on that engine, and a settled turn on one engine starts no other engine (with `@codex` installed, the codex adapter is not spawned by a claude turn settling; assert on the scripted adapter's spawn log the way the existing "started to answer, asked, and stopped" scenarios do); the scenario that today forbids a second listing after an already-listed conversation settles is rewritten to assert that narrower rule instead; an agent that could not be asked is named in the log and files nothing.

### 5.10 Docs

Same PR. Rewrite, do not append:

- `docs/chat.md`: `## Node agents` and everything under it, `## Which conversation you come back to` (the note records nothing new; reload folds everything and the sidebar's Recent is how you get back), `## Who, and which model, the header names` (now "the agent line"), the `sessions (n)` paragraphs (retired, say why) and `+ new` (now mints an Inbox node), `## Moving the chats you already have` (the filer, the `Chats` node, the first commit's size, `Move to…` as reassignment, trash is final, nesting, the second machine), `## Asking about one node` (nearest ancestor agent), phone paragraphs. `docs/plugins/capture.md` (or wherever the Inbox is documented): the `Chats` node and the service capture declares.
- `docs/plugins/chat.md`: seat table (`## Where it hangs in the tab`) with the four new slots (`outline.row.aside`, `outline.row.fold`, `outline.page.head`, `outline.page.foot`) and the removed `app.panel`/`app.header` seats.
- `docs/architecture/e2e-coverage.md`: the node-agent scenario inventory; `docs/architecture/e2e-economy.md:46` if the unit/e2e split for scopes moved.
- `docs/architecture/overview.md:181` section naming the roster.
- `docs/images/acp/*.png` referenced from `docs/chat.md:70`: retake the three screenshots on the new page.
- `website/`: only if a picture there shows the right dock; check `website/index.html` images before touching anything.

## 6. Cordis checklist for the reviewer

- [ ] Every fold owns its subscription and releases it on fold, row removal, navigation, and plugin runtime rebuild (test the rebuild path the way `chat_*` features test "the plugin runtime rebuilds").
- [ ] No module-level signal describes a conversation; `refused`, drafts, dismissed completions, holding, asked are per `Chat`.
- [ ] The three slot faces are registered under `AgentsProvider` and nothing else; no face reaches `chatWire()` for the roster.
- [ ] Layout's `shell` service is no longer named by chat, and no chat file imports from `olai-plugin-layout/contract` for the panel.
- [ ] Withdrawal order: unregistering `outline.row.fold` while a fold is open disposes the fold's owner before the slot leaves (see how `Doors.tsx` handles a door leaving with its row: `a_door_leaves_with_its_row.feature`).
- [ ] Optional availability: with no engine installed, the aside draws nothing and the pill is absent; with the search plugin off, `Ask agent`'s ancestor lookup still works (it reads the outlines collection, not search).
- [ ] Reconnection: a server restart re-subscribes every open fold and re-draws its transcript from the replay, the way the dock did (`node_agent_connection.feature`).
- [ ] Atomic ownership: two tabs unfolding the same node are two readings of one scope; neither can evict the other.
- [ ] The filer is owned by the chat server scope, names capture's inbox service optionally, runs on events only, and is stopped with the scope; a run in flight when the scope closes is interrupted, not left writing.
- [ ] Several folds open at once are several owners; disposing one never touches another's subscription.

## 7. Out of scope, noted for the next PR

- Merging node recency and agent recency into one "last activity" on the roster row (server-side join), if Recent ordering proves surprising.
- Agency (a node agent creating child agents) and relocating the scheduler behind its own plugin, per `docs/chat.md:739-744`.
