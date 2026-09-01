# Node agents: an agent associated with an olai node

*The human's reframe, 2026-09-01, in conversation with the orchestrator; terminology ruled the same evening — "node agents," plain words ("bot" survives only Spaces-side, where a bot user is Spaces' own term; "seat" was considered and rejected as cryptic). Supersedes the chats↔nodes↔notes association triangle (`docs/brainstorming/associations.md`): notes drop out entirely, and the node stops being something a chat is ABOUT and becomes the thing the chat belongs to. UI prototype: `projects/olai/prototypes/node-agents-roster.html`.*

## The idea in one sentence

**Creating a node agent is creating an olai node**: put an `agent` prop on any node and an agent is associated with it — the node's title is its name, its desc is its charter, its SUBTREE is its memory. An ACP chat session gets associated with the node and can be thrown away and recreated at any time, because the subtree — not the transcript — is what the agent knows.

## Why this is already proven

The orchestrator has run exactly this model by hand since August: a conversation whose durable memory is a subtree (the boot chain + the board), whose sessions die, compact and get recreated, and whose standing law says *session memory dies with the session; the board is the memory; a fresh session must be able to read the board and know everything a dead session knew.* Node agents are that discipline turned into product.

## The three-way split the word makes plain

| thing | word | lives where |
|---|---|---|
| the **engine** | the roster's agents (claude, grok, pi) | a process, swappable |
| the **node agent** | node + `agent` prop: name, charter, subtree-memory | the board, durable |
| the **session** | a chat (or a terminal) | cattle, recreated freely |

**The `agent` prop collision dissolves rather than needing a second key**: on a lane and on a standing node alike, `agent` means "the engine currently associated with this node." A dispatched PR lane was always a *temporary* node agent whose session-vehicle is a kolu terminal; a standing one's vehicle is a chat. The future orchestrator — an agent that creates child nodes and associates agents with them — is what today's orchestrator already does five times a day when it dispatches lanes; this makes it product.

## What falls out for free

1. **Wake scope dissolves as a setting.** A node agent's doorbell scope IS its subtree: it wakes on terminals and CI runs claimed under it, nothing else. Derived, never configured — the #457 picker question evaporates for node agents.
2. **"Ask agent" gains a routing rule.** Asking any row routes to the NEAREST ANCESTOR node agent, with the row as the chip. One rule, no picker.
3. **Layer 3 maps one-to-one.** A node agent ↔ a Spaces bot identity/thread; the mirror binds the NODE, so it survives session churn.

## The panel, re-founded (as ruled + prototyped)

- **The roster lives in the LEFT SIDEBAR** (ruled): an AGENTS section — one row per node agent with its engine, live state dot (working / needs-you / idle / asleep) and unread count. The roster is literally the query `prop:agent`.
- **On the outline, an agent-carrying row wears a door + its last message** (ruled): the kolu Dock-row pattern — state, session, memory size, and one line of the latest message.
- **Pressing the door or a roster row switches the RIGHT panel to that agent** (ruled): the outline keeps its width; the panel's header names the NODE first (pressable), engine+model second, with `sessions (n)` and a **fresh session** affordance labeled with what it means ("memory is the subtree; the transcript becomes history").
- Past chats demote to a per-agent detail: that agent's sessions.

## Migrating existing chats (the human's question, 2026-09-01)

Migration is **association, not conversion** — nothing moves on disk; session files stay wherever their agent keeps them.

1. **Old chats wait under an UNASSIGNED roster entry** (the human's ruling, same evening): the sidebar's AGENTS section ends with an "Unassigned (n)" row holding every conversation no node claims. Opening it lists them in the panel, each with **"assign to node…"** (`@` node-completion). Assigning writes the node↔session pointer — and if the target node carries no `agent` prop yet, sets it to the chat's own engine. The old conversation becomes the node agent's CURRENT session, context intact — a home, not an abandonment.
2. **One distillation turn banks the memory.** An old chat's knowledge lives only in its transcript, and the contract says the subtree is the memory — so an assigned session's first order is: *your memory is now this subtree; write your standing facts into it.* After that turn, the session is cattle like any other. (The orchestrator conversation is the degenerate case: its banking discipline ran all along, so its distillation is a no-op.)
3. **Supersession lineage rides along.** The panel already infers /clear chains (which conversation replaced which); assigning a chat claims its chain as the node agent's session HISTORY, so "past sessions (n)" is populated from day one.
4. **Incremental by construction.** Unassigned chats keep working exactly as today — a subset migrates one row at a time; there is no flag day, and Unassigned may simply never empty.

Honest note: the association is MACHINE-LOCAL (session ids do not travel between machines — the same reason the which-conversation note is per-machine state), while the `agent` prop is board-durable. A vault served from two machines gets one node agent with per-machine sessions, and the subtree-memory is what keeps them coherent.

## The three hard parts

1. **Write-back discipline is the whole trick.** Subtree-as-memory only works if the agent WRITES standing facts into its subtree — the orchestrator's law, re-read every boot. For arbitrary node agents it must be product: the boot/system prompt teaching "your subtree is your memory; write what a successor needs." **Ruled: this teaching ships INSIDE phase 1**, scoped to agent-associated sessions (the broader acp-system-prompt item stays its own thing).
2. **Nested agents.** A node agent on a project and one on a lane beneath it share ground. Reads route by nearest-ancestor; writes need a rule — simplest: an agent writes only strictly inside its own subtree and ASKS its ancestor for anything above (the orchestrator's own stop-and-ask, generalized).
3. **Session economics.** N node agents are not N processes: sessions spawn lazily on first message (or first wake in scope) and reap when idle; the roster chip stays honest about asleep-vs-broken — the kolu pill's three-state pattern.

## The phases (ruled with the human, 2026-09-01, question tool)

1. **Roster + switching.** The AGENTS sidebar section over `prop:agent`, the door + last message on agent rows, the right panel switching on press — and the subtree-memory teaching for agent-associated sessions (the keystone, ruled into this phase). Sessions bind via config/state; no assign UI, no new wake logic. Ships something visible and honest: the panel re-founded, working over hand-bound sessions.
2. **Migration.** The Unassigned roster entry, the assign-to-node gesture, the distillation order, supersession lineage claiming, the fresh-session affordance. After this phase a person can move their real chats over, one row at a time.
3. **Derived wakes.** A node agent's doorbell scope becomes its subtree, replacing the manual wake control for agent-bound conversations; **an agentless wake CLIMBS to the nearest ancestor node agent** (ruled) — the orchestrator ends up the root catcher, and nothing is silently lost. The manual control survives only for unassigned conversations.
4. **Agency.** A node agent creates child nodes and associates agents with them — dispatch as product (create node, set `agent`, seat a session or a terminal), with the writes-stay-inside-the-subtree rule and stop-and-ask to the ancestor. The layer-3 binding (a node agent ↔ a Spaces bot/thread) rides here too.
5. **The orchestrator migrates LAST** (ruled): once assign, distill, derived wakes and agency have proven out on smaller agents, the orchestrator conversation is assigned to the orchestrator node — the no-op distillation — and the manual boot-chain era ends as product catches up with practice.

## Still open for the design round

- Which engine CLIs can hold many concurrent sessions per directory cleanly (Claude sessions are per-directory files; grok/kimi session models need the same check) — a fact-check, not a ruling.
