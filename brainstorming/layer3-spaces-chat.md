# Layer 3: team chat on Xyne Spaces, backed by each team's Kolu+Olai

*The human's three-layer frame (2026-09-01): L1 kolu — filesystem/terminals/agent CLIs, manual prompting; L2 olai — git-backed memory + the one orchestrator; L3 — multi-player chat on Xyne Spaces over team-hosted Kolu+Olai. Fact-grounded from the xyne-spaces repo, 2026-09-01: three parallel researcher sweeps, then the load-bearing claims RE-VERIFIED first-hand by the orchestrator (each carries its citation; the re-check corrected one researcher error, noted below).*

## The deployment facts that frame everything

- Spaces is ONE org-hosted multi-tenant deployment; the repo ships no per-team/self-host shape (compose files are dev/CI only; no helm/installer — NOT FOUND). Teams host **kolu+olai only**, so everything the team initiates must go **outbound from the team box**.
- Spaces' agent plane (xyne-claw) runs agents in *Xyne's* Kata sandboxes (`claw-deployments/`), tied to Spaces via `Agent.spacesAppId/spacesAppToken` (claw-auth `schema.prisma`). **A team-hosted runner that dials into Spaces: NOT FOUND.** External CLIs reach *in* (`packages/xyne-claw-mcp`, `packages/xyne-claw-pi-plugin`) — the opposite direction.

## What Spaces supports today — verified first-hand

- Bots are first-class: `userType String @default("USER") // USER or BOT` (`apps/backend/prisma/schema.prisma:814`); `Message.msgType` `USER|BOT|SYSTEM|FORWARDED`.
- **Installed-app REST, write**: `router.post('/postMessage', requirePermission('chat:write'), validateChannelAccessForPost, ...)` (`apps/backend/src/apps/routes/chat.ts:9`), behind `authenticateApp` (per-app JWT signed with `Apps.signingSecret` — `schema.prisma:4079`). App-token messages persist as BOT; "an app token can never forge a message that appears to come from a human" (`chatController.ts:284-292`, verbatim comment; only the internal S2S `postAsUser` route may author as a human).
- **Installed-app REST, read**: `router.get('/channelHistory', requirePermission('channels:read'), ...)` and `conversationReplies` (`apps/routes/chat.ts:12-13`; controller at `chatController.ts:509-511`, `GET /api/external-event/chat/channelHistory?channelId=…&limit=…&cursor=…`). **An externally hosted bridge can READ channels today** — this corrects the initial researcher report, which called it NOT FOUND.
- **Incoming webhooks**: `POST /api/apps/webhooks/:workspaceId/:appId/:secret` → `findOrCreateConversation(..., MessageType.BOT, ...)` (`incomingWebhookController.ts:313-322`).
- **Ephemeral progress**: `POST /api/apps/chat/agentProgress` (same route file, `chat:write`) → SYSTEM event over Redis→Socket.IO.
- Delivery to humans: Socket.IO (`websocketService.ts:75-83`) + Redis pub/sub; rocicorp Zero client sync.

## What Spaces does NOT support today — verified first-hand

**Push delivery of channel messages to an externally hosted bot.** The bots that answer @mentions are in-process implementations behind a hardcoded allowlist — `CHAT_ENABLED_BOT_IDS = new Set(['ask-ai'])`, with the comment "Add a bot ID here" (`apps/backend/src/services/bots/chat-enabled-bots.ts:6-8`) — and the only outbound POSTs to external URLs are per-button action callbacks (`dispatchAction`, SSRF-guarded). The claw-auth `mcpgateway` explicitly has "no public HTTP execution endpoint. Tool execution is invoked internally" (`docs/mcp-gateway-integration.md:54`).

## The phases, corrected

1. **Mirror (olai plugin only, no Spaces changes).** `plugin-xyne-spaces` holds an installed-app token and posts orchestrator replies + doorbell deliveries (the #450 seam already yields them as messages) via `postMessage`, with `agentProgress` while a turn runs. Multi-player *visibility*.
2. **Two-way by polling (still no Spaces changes).** The same plugin polls `channelHistory`/`conversationReplies` (`channels:read`) with a cursor and relays new human messages into the conversation as attributed input (`senderId` rides the record; olai's delivery seam accepts composed inputs). Latency = poll interval.
3. **Two-way by push (one Spaces PR).** An "external bot" arm on the unified bot framework: message/mention in a bound channel → POST to the app's registered URL, HMAC-signed with the existing `Apps.signingSecret`, through the existing SSRF guard — the Slack Events API shape from parts already in the repo. Requires the team box reachable from Spaces (VPN/tunnel) — the same constraint `mcpgateway` accepts for services it calls back.
4. **Org memory (separable).** Transcripts/lane events into the Spaces context store as a connector; and optionally team-olai tools registered into claw-auth's `mcpgateway` (S2S-key-gated) so Spaces' own Ask-AI agents can query a team's board.

## Open rulings (matter from phase 2 on)

1. **Authority**: who in the channel counts as "the human" for gates (merge-human, deferrals)? Simplest: anyone with channel write — the team collectively — attributed per message via `senderId`.
2. **Channel granularity**: one channel per team-orchestrator, or per lane? (The doorbell digest compresses well; per-lane rooms multiply.)
3. **Poll cadence vs push timing**: whether phase 2 (polling) ships at all or waits for phase 3's push, given the latency a steering surface tolerates.
