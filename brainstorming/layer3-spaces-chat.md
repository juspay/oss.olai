# Layer 3: your team's AI fleet, in your team's chat

**The idea in one sentence:** every team runs its own Kolu+Olai box (layer 1+2); Xyne Spaces gives the whole org multi-player chat; layer 3 connects them so a team's channel becomes the place where the fleet reports and takes orders.

*Grounded in the xyne-spaces source, 2026-09-01: three research passes plus a fourth exhaustive Apps-framework sweep, load-bearing claims verified first-hand. Details and citations live in the footnotes.*

## What a team member gets

- **A channel where the fleet talks.** "CI went green on #97." "PR #456 merged." "The author on lane X is stuck." Every doorbell event the orchestrator receives shows up in the team channel, live, with a "working…" indicator while it runs a turn.
- **Talking back.** @mention the bot ("@olai merge #97") or DM it — Spaces pushes the message to the team's box, signed; the orchestrator acts and replies in-thread.
- **Slash commands.** "/olai status", "/olai dispatch …" — registered per app, dispatched straight to the team box with channel/user context.
- **Real buttons on real questions.** Merge gates as interactive screens: the orchestrator posts "merge #97? [merge] [park]", a click round-trips to the box, the screen updates in place.
- **Everything is ordinary chat**: attributed, threaded, searchable, permission-scoped by Spaces.

## The user flows

1. **Watching.** CI settles → doorbell fires → the bridge posts to the channel seconds later.[^post]
2. **Steering by mention/DM.** "@olai ship it" → Spaces POSTs a signed `APP_MENTIONED` event (sender, channel, clean text) to the team box → the orchestrator conversation receives "«Sridhar»: ship it" → acts → the bridge posts the reply.[^events]
3. **Commands.** "/olai status" → Spaces POSTs the command payload (command name, text, channel, user) to the box; the bridge answers by posting.[^commands]
4. **Gates as buttons.** The bridge posts a Flow screen; a click POSTs the action (signed) to the box synchronously; the bridge returns the next screen — "merged ✓" — which replaces the message in place.[^flow]

## The architecture, high level

```
 org-hosted                            team-hosted (one per team)
┌──────────────────┐  events/commands ┌──────────────────────────┐
│  Xyne Spaces     │ ────────────────▶│  bridge (an olai plugin) │
│  channel + bot   │   (HMAC-signed)  │     ↕ conversation       │
│  (installed app) │ ◀────────────────│  OLAI orchestrator       │
└──────────────────┘   post/update/   │     ↕ terminals          │
                       read via REST  │  KOLU fleet              │
                                      └──────────────────────────┘
```

- One **Spaces app per team**: a bot user per workspace, a webhook URL pointing at the team box, scoped permissions, versioned installs.[^app] The existing "kolu" app already runs this over a tailnet URL — reachability solved.
- One **olai plugin** (`plugin-xyne-spaces`) team-side: posts outward with the app JWT, verifies `X-Xyne-Signature` on inbound events, feeds them into the orchestrator conversation through the doorbell delivery seam.[^plugin]
- **Nothing moves.** Orchestrator, vault, terminals stay on the team box; Spaces stores only the chat.

## What Spaces already ships (all verified)

| Need | Shipped mechanism |
|---|---|
| Bot posts to channel | installed-app REST, `chat:write`, Block-Kit attachments accepted[^post] |
| Bot edits its messages | `updateMessage`, same scope[^post] |
| Live "working…" | `agentProgress` ephemeral signal[^post] |
| Bot reads history (catch-up) | `channelHistory` / `conversationReplies`, `channels:read`, cursored[^post] |
| Push on @mention / DM | `APP_MENTIONED` / `DIRECT_MESSAGE` events to the app webhook, HMAC-signed[^events] |
| Slash commands & shortcuts | per-app registry, dispatched to the webhook with context[^commands] |
| Interactive screens | Flow UI: post a screen, button clicks round-trip synchronously, app returns the next screen[^flow] |
| Slack-SDK compatibility | a Slack Web-API-shaped adapter for the bot's *inbound* calls[^slack] |

**So the whole three-phase feature set needs zero Spaces PRs.** The bridge is olai-side work only.

## Honest limits (also verified)

- **No firehose.** A channel message that mentions nobody never reaches an app webhook — by design, confirmed in the message write path. Steering is mention/DM/command-shaped, which fits how bots are used anyway; the bridge can cursor `channelHistory` if it ever needs catch-up context.[^limits]
- **Event delivery is fire-and-forget** — failures are logged, not retried. The bridge treats polling as its safety net for missed events.[^limits]
- **Command dispatch is not HMAC-signed today** (mention/DM events and Flow actions are). Until that's hardened upstream, the bridge treats unsigned command payloads as untrusted: fine for read-only commands ("/olai status"), not for privileged orders — those ride mentions or buttons, which are signed. This is the one candidate for a small upstream Spaces PR, security-shaped, optional.[^limits]

## The phases (all olai-plugin work)

1. **Mirror** — post doorbell events + orchestrator replies; `agentProgress` during turns.
2. **Steer** — receive mention/DM events, verify signature, relay attributed into the conversation.
3. **Polish** — register `/olai …` commands; Flow screens for gates; optionally the org-context connector so lane history is org-searchable.

## Open rulings for the human

1. **Authority** — anyone with channel write counts as "the team's word" (attributed per message), or a named allowlist?
2. **Granularity** — one channel per team-orchestrator, threads per lane?
3. **What mirrors** — every doorbell event, or digest level (settles, merges, blocks)?

---

[^app]: `Apps` model: `webhookUrl`, app-level `signingSecret`, scoped permissions, versioned — `apps/backend/prisma/schema.prisma:4069-4120`. Install is per-workspace with a bot user minted per `(app, workspace)` (`appUtils.ts:118-153`); re-install is the "Update" that re-syncs commands/permissions (`appUtils.ts:77-178`). Webhook URL editable per install (`PATCH /api/apps/installed/:id`) or on the template ("installs pick it up on Update" — `EditAppForm.tsx:1415`). 13 permission scopes exist (`seed-app-permissions.ts:9-23`); the bridge needs `chat:write`, `channels:read`, maybe `im:write`, `files:*`.

[^plugin]: The delivery seam is olai #450's (the kolu doorbell); the odu doorbell lane in flight reuses it. The plugin's server half holds the app JWT + signing-secret verification and an HTTP listener for the webhook (the team box is reachable via tailnet, as the existing kolu app proves).

[^post]: Write: `POST /api/apps/chat/postMessage` (`routes/chat.ts:9`, `chat:write`) — accepts text, markdown, Slack Block-Kit `attachments[]` (`chatController.ts:171-174`), and Flow screens; BOT-typed, cannot forge humans (`chatController.ts:284-292`). Update: `.../updateMessage` (`chat.ts:10`). Progress: `.../agentProgress` (`chat.ts:11`, ephemeral, Redis→Socket.IO). Read: `channelHistory`/`conversationReplies`/`conversationAttachments` (`chat.ts:12-14`, `channels:read`).

[^events]: Dispatcher `apps/backend/src/apps/core/eventSubscriptionUtils.ts`: JSON event + `X-Xyne-Signature` (HMAC-SHA256, app secret) + `X-Source: XyneSpaces`, SSRF-guarded, `redirect:'manual'`. Kinds: `APP_MENTIONED`, `DIRECT_MESSAGE`, `USER_MENTIONED`, `EMAIL`, `ADDITIONAL_FORM_FIELD_UPDATED`, `DESK_REPLY` (`apps/types/index.ts:28-35`). Payloads carry `userId`, `senderName`, `cleanContent`, channel/conversation/message ids, attachments (`:55-91`). Mention emission: `messages-handler.ts:650-674` (channel), `:1680-1699`/`:1846-1866` (group-DM threads/top-level); DM: `:1581-1603` (app must be a DM participant); the sender's own app is always excluded.

[^commands]: Models `AppCommand`/`InstalledAppCommand` (template vs frozen install snapshot, re-synced on Update — `appUtils.ts:23-71`). Invocation `POST /api/apps/channel/:channelId/command` (`commandController.ts:492-705`): POSTs `{actionId, type: command|shortcut|message_shortcut, values:{text}, context:{channelId, conversationId, userId, messageId, flowJSON?}}` with `X-Xyne-Event` header, 30s sync timeout; errors surface to the invoking user as ephemeral notices.

[^flow]: Post a screen via `postMessage` with `flow:{version:'2.0', screenId, title, components[], …}`; a component action hits `POST /api/apps/flow/action` (`flowController.ts:32-215`) which POSTs to the app HMAC-signed and, on an `open_screen`/`next_screen` response, renders the returned screen in place — a synchronous interactive round trip. Buttons outside Flow use `dispatchAction` (fire-and-forget, `chatController.ts:603-678`).

[^limits]: No-mention-no-webhook confirmed at `messages-handler.ts:640-699` (empty mention set ⇒ zero app-event branches fire). Fire-and-forget fan-out: `eventSubscriptionUtils.ts:89-108` (`.map(async…)` uncollected, failures logged only). Unsigned command dispatch: no `signWebhookPayload` call in `commandController.ts` (contrast `flowController.ts:111-148`); `prCheckCallback.ts`'s dispatch is likewise unsigned. Also: Spaces ships no per-team self-host (dev/CI compose only) and its claw agent plane runs in Xyne's own Kata sandboxes with no team-hosted-runner concept — which is exactly why the bridge lives on the team box and dials/receives as an installed app instead.
