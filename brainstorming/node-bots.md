# Node bots: an olai node becomes a first-class chat agent

*The human's reframe, 2026-09-01, in conversation with the orchestrator. Supersedes the chats↔nodes↔notes association triangle (`docs/brainstorming/associations.md`): notes drop out entirely, and the node stops being something a chat is ABOUT and becomes the thing the chat IS. UI prototype: `projects/olai/prototypes/node-bots-roster.html`.*

## The idea in one sentence

**Creating an agent bot is creating an olai node**: put an `agent` prop on any node and it is a bot — its title is the bot's name, its desc is its charter, its SUBTREE is its memory. An ACP chat session gets associated with the node and can be thrown away and recreated at any time, because the subtree — not the transcript — is what the bot knows.

## Why this is already proven

The orchestrator has run exactly this model by hand since August: a conversation whose durable memory is a subtree (the boot chain + the board), whose sessions die, compact and get recreated, and whose standing law says *session memory dies with the session; the board is the memory; a fresh session must be able to read the board and know everything a dead session knew.* Node bots are that discipline turned into product — everyone gets what the orchestrator has.

## The core, olai-shaped

- **The bot is a prop, not a new thing.** `agent` is already a typed roster-ref key. A node carrying it is a bot; taking it off retires the bot with the memory left readable. One board write either way — decision-shaped, exactly the kind of fact props exist for.
- **Sessions are cattle.** The panel keeps node → current-session-id in machine state (the same state-dir note that today remembers which conversation the panel holds). "Fresh session" is safe by construction: the new session's first act is reading its own subtree. `/clear` stops meaning memory loss.
- **The charter is the desc.** What the bot is FOR lives on the node itself, versioned in git like everything else.

## What falls out for free

1. **Wake scope dissolves as a setting.** Today a conversation's doorbell scope is a manually picked file (#457's whole saga). A node bot's scope IS its subtree: it wakes on terminals and CI runs claimed under it, and on nothing else. Derived, never configured — the picker question evaporates for bots.
2. **"Ask agent" gains a routing rule.** Asking any row routes to the NEAREST ANCESTOR BOT, with the row as the chip. One rule, no picker: rows under a project node with a bot go to that bot; everything else climbs to the orchestrator.
3. **Layer 3 maps one-to-one.** A node bot ↔ a Spaces bot identity/thread. The Spaces mirror binds the NODE, not a session — so it survives session churn, which is exactly what a channel binding should do.

## The panel, re-founded

"A list of chats with one agent" becomes **a roster of bots**:

- **Roster list**: one row per bot — node title, agent mark, a live state chip (working / waiting-on-you / idle / asleep), unread count. Reads like a DM list. The roster is literally the query `prop:agent` — the board and the roster are two views of one set.
- **A bot's view**: header names the BOT first (pressable — jumps to its node), agent+model second; transcript and composer as today; a **fresh session** affordance labeled with what it means ("memory is the subtree; this transcript becomes history").
- **Past sessions** demote to a per-bot detail — "this bot's sessions", the old chats list one level down.
- **On the outline**, a bot node's row wears a **chat door** (the terminal-door pattern): a small face showing the bot's state, pressable to open its panel.
- **"+ new"** becomes **new bot**: pick or create a node, pick an agent from the roster. The agent choice moves off the conversation and onto the node, durable.

## The three hard parts (to rule before any lane)

1. **Write-back discipline is the whole trick.** Subtree-as-memory only works if the bot WRITES standing facts into its subtree. The orchestrator does this by vault law it re-reads every boot; arbitrary bots need it as product — the boot/system prompt that teaches "your subtree is your memory; write what a successor needs" is the parked `acp-system-prompt` roadmap item, promoted to the keystone of this feature.
2. **Nested bots.** A bot on a project node and a bot on one of its lanes share ground. Reads route by nearest-ancestor; writes need a rule — simplest: a bot writes only strictly inside its own subtree and ASKS its ancestor for anything above (the same shape as the orchestrator's stop-and-ask).
3. **Session economics.** N bots are not N processes: sessions spawn lazily on first message (or first wake in scope) and reap when idle; the roster chip is honest about asleep-vs-broken, the same three-state honesty the kolu pill established.

## Open questions parked for the design round

- Does the orchestrator itself become "just" the bot on the orchestrator node on day one, or does it migrate last, after the pattern is proven on smaller bots?
- Which agent CLIs can hold many concurrent sessions per directory cleanly (Claude sessions are per-directory files; grok/kimi session models need the same check)?
- Does a wake for a subtree with NO bot climb to the nearest ancestor bot, or stay silent? (Climbing makes the orchestrator the default catcher — probably right, and it matches Ask-routing.)
- The `agent` prop is claimed today by lanes to mean "who authors this PR" — the same key meaning "who inhabits this node" is a collision to resolve (a second key like `bot`, or a ruling that the meanings are one).
