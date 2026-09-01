# Layer 3: your team's AI fleet, in your team's chat

**The idea in one sentence:** every team runs its own Kolu+Olai box (layer 1+2); Xyne Spaces gives the whole org multi-player chat; layer 3 connects them so a team's channel becomes the place where the fleet reports and takes orders.

*Grounded in the xyne-spaces source, 2026-09-01, twice-corrected by deeper reads — details and file citations in the footnotes. A full Apps-framework sweep is in flight; this doc absorbs its findings when it lands.*

## What a team member gets

- **A channel where the fleet talks.** "CI went green on #97." "PR #456 merged." "The author on lane X is stuck and needs a word." Every doorbell event the orchestrator already receives shows up in the team channel, live. Nobody has to open olai or ssh anywhere to know how the work is going.
- **Talking back.** @mention the orchestrator's bot in the channel ("@olai merge #97", "@olai what's blocking the doorbell lane?") or DM it. It answers in-thread, and acts — because behind the bot is the same orchestrator conversation that runs the fleet today.
- **Everyone sees everything.** The whole exchange is ordinary team chat: attributed to whoever spoke, threaded, searchable, permission-scoped by Spaces like any other channel.

## The user flows

1. **Watching (passive).** CI settles on a lane → the orchestrator's doorbell fires → the team channel shows "odu #97: green, both platforms" seconds later. A red shows up the moment the first check fails.
2. **Steering.** A teammate types "@olai ship it" on that thread → Spaces pushes the mention to the team's box → the orchestrator gets "Sridhar said: ship it" in its conversation → merges → replies in the thread. Who-said-what comes from Spaces' own identity; the orchestrator's law ("the human's word") becomes "the team's word, attributed."
3. **Asking privately.** DM the bot → same loop, private thread.
4. **Working while it works.** While the orchestrator runs a turn, the channel can show a live "working…" signal rather than silence.

## The architecture, high level

```
 org-hosted                          team-hosted (one per team)
┌─────────────────┐   events out    ┌──────────────────────────┐
│  Xyne Spaces    │ ──────────────▶ │  bridge (an olai plugin) │
│  team channel   │  mention / DM   │     ↕ conversation       │
│  + bot user     │ ◀────────────── │  OLAI orchestrator       │
│                 │   post message  │     ↕ terminals          │
└─────────────────┘                 │  KOLU fleet              │
                                    └──────────────────────────┘
```

- Each team registers **one Spaces app** = a bot user + a webhook URL pointing at the team's box.[^app] The screenshot-proven "kolu" app already does exactly this over a tailnet URL, which also answers reachability.
- The team side is **one olai plugin** (`plugin-xyne-spaces`), the same shape as the kolu and odu plugins: it posts outward with the app's token, and receives Spaces' signed event deliveries, feeding them into the orchestrator conversation through the delivery seam the doorbells already use.[^plugin]
- **Nothing moves.** The orchestrator, its memory (git vault), and the terminals stay on the team box. Spaces stores only what any chat stores: the messages.

## Why this is cheap: Spaces already has both directions

**Outbound from the team (posting):** shipped today. An installed app posts messages as its bot via REST, fires a live "working…" signal, and can even read channel history with a cursor.[^post]

**Inbound to the team (push):** shipped today, contrary to this doc's first two drafts. Spaces pushes **APP_MENTIONED** and **DIRECT_MESSAGE** events (plus user-mention, email, desk-reply kinds) to the app's registered webhook URL, HMAC-signed, fired from the message write path.[^events] What has NOT been found (pending the full sweep): delivery of ordinary un-mentioned channel messages.

So the earlier "one Spaces PR to close the gap" is likely **zero Spaces PRs** for the core loop: mention-and-DM steering is enough to start, and matches how people use bots anyway.

## The phases

1. **Mirror** — the plugin posts doorbell events + orchestrator replies to the channel. Team watches. *(olai plugin only.)*
2. **Steer** — the plugin receives mention/DM events and relays them, attributed, into the conversation. *(olai plugin only, if the sweep confirms the event contract suffices.)*
3. **Polish** — slash commands ("/olai status"), interactive buttons on gate questions ("merge? [yes] [park]"), org-context connector so lane history is searchable org-wide. *(Scope depends on the sweep's findings on commands/blocks.)*

## Open rulings for the human

1. **Authority** — anyone with channel write counts as "the human," attributed per message? Or a named-users allowlist?
2. **Granularity** — one channel per team-orchestrator (lean), threads per lane?
3. **What mirrors** — every doorbell event, or the digest level (settles, merges, blocks — not every heartbeat)?

---

[^app]: `Apps` model: `webhookUrl`, `signingSecret`, scopes, versioned installs — `apps/backend/prisma/schema.prisma:4069-4120`. The edit dialog (Basic info / Commands / Shortcuts / Permissions) is the human's screenshot; "installs pick it up on Update" is its own caption.

[^plugin]: The delivery seam is olai PR #450's (the kolu doorbell); the odu doorbell lane in flight reuses it. A plugin's server half already holds outbound connections and composes conversation messages.

[^post]: Write: `POST /api/apps/chat/postMessage` behind per-app JWT + `chat:write` scope (`apps/backend/src/apps/routes/chat.ts:9`); messages persist as BOT and "an app token can never forge a message that appears to come from a human" (`chatController.ts:284-292`). Live signal: `POST .../agentProgress` → SYSTEM event over Redis→Socket.IO. Read: `GET .../channelHistory` + `conversationReplies` behind `channels:read` (`routes/chat.ts:12-13`, controller `:509`). Incoming URL-secret webhooks also exist (`incomingWebhookController.ts:313-322`).

[^events]: Dispatcher: `apps/backend/src/apps/core/eventSubscriptionUtils.ts` — `sendWebhookNotification` POSTs the JSON event with `X-Xyne-Signature` (HMAC-SHA256, app's `signingSecret`), SSRF-guarded via `prepareAppWebhookDispatch`, `redirect: 'manual'`. Event kinds: `AppEventType` enum — `APP_MENTIONED`, `DIRECT_MESSAGE`, `USER_MENTIONED`, `EMAIL`, `ADDITIONAL_FORM_FIELD_UPDATED`, `DESK_REPLY` (`apps/backend/src/apps/types/index.ts:28-35`). Emission sites include the message write path (`zero/side-effects/tables/messages-handler.ts:2326,2377`) with the sender's own app excluded. Deployment facts that frame the topology: Spaces ships no per-team self-host (dev/CI compose only); the claw agent plane runs in Xyne's Kata sandboxes — so teams host kolu+olai, and the bridge's inbound leg needs the box reachable from Spaces (the existing kolu app solves this with a tailnet URL).
