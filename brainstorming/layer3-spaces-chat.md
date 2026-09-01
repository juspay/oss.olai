# Layer 3: team chat on Xyne Spaces, backed by each team's Kolu+Olai

*The human's three-layer frame (2026-09-01): L1 kolu — filesystem/terminals/agent CLIs, manual prompting; L2 olai — git-backed memory + the one orchestrator; L3 — multi-player chat on Xyne Spaces over team-hosted Kolu+Olai. This doc records the fact-grounded design from the 2026-09-01 session; every claim below was read out of the xyne-spaces repo that day (paths cited), gaps marked as gaps.*

## The deployment facts that frame everything

- Spaces is ONE org-hosted multi-tenant deployment; the repo ships no per-team/self-host shape (compose files are dev/CI only; no helm/installer). Teams host **kolu+olai only**. So the bridge must originate **outbound from the team box** for everything the team box initiates.
- Spaces' agent plane (xyne-claw) runs agents in *Xyne's* Kata sandboxes, tied to Spaces via `Agent.spacesAppId/spacesAppToken`. **A team-hosted runner that dials into Spaces does not exist** (searched: nothing). External CLIs reach *in* (`packages/xyne-claw-mcp`, `packages/xyne-claw-pi-plugin`) — the opposite direction.

## What Spaces already supports (the team→Spaces half — buildable today)

- Bots are first-class: `User.userType: "BOT"` (`apps/backend/prisma/schema.prisma:814`); `Message.msgType` `USER|BOT|SYSTEM|FORWARDED`.
- **Installed-app REST**: `POST /api/apps/chat/postMessage` behind `authenticateApp` — per-app JWT signed with `Apps.signingSecret`, Slack-style scopes (`chat:write`); posts as the app's bot user; app tokens cannot forge a human `senderId` (`chatController.ts:284-292`).
- **Incoming webhooks**: `POST /api/apps/webhooks/:workspaceId/:appId/:secret` → BOT message into the bound channel (`incomingWebhookController.ts:272-334`).
- **Ephemeral progress**: `POST /api/apps/chat/agentProgress` → SYSTEM event over Redis→Socket.IO (the typing-indicator channel).
- Delivery: Socket.IO (`websocketService.ts:75-83`) + Redis pub/sub; rocicorp Zero for client sync.

**Therefore phase 1 is an olai plugin and nothing else**: `plugin-xyne-spaces` holds the installed-app token, posts orchestrator replies + doorbell deliveries (the #450 seam already produces them as messages) to the team channel, and `agentProgress` while a turn runs. One-way, multi-player *visibility* — the team watches the fleet from Spaces.

## The one real gap (the Spaces→team half)

**No outbound message delivery to an externally hosted bot exists.** The bots that answer messages are in-process implementations (`apps/backend/src/bots/implementations/*`); the @mention allowlist is hardcoded to `ask-ai` (`chat-enabled-bots.ts`); the only outbound calls to external URLs are per-button action callbacks (`dispatchAction`, SSRF-guarded). Also NOT FOUND: any app-token API or socket subscription for *reading* channel messages.

**Phase 2 is one Spaces PR**, assembled from parts already in the repo: an "external bot" arm on the unified bot framework — when a bound channel receives a message (or the app's bot is @mentioned), POST it to the app's registered URL, HMAC-signed with the existing `Apps.signingSecret`, through the existing SSRF guard. The Slack Events API shape. Attribution rides free: the delivered message carries `senderId`, so olai receives who-said-it from Spaces' identity, and olai's conversation-delivery seam accepts composed inputs unchanged.

Constraint to accept: the team box must be reachable from Spaces (VPN/tunnel) — the same constraint the repo's own `mcpgateway` accepts for services it calls back (`docs/mcp-gateway-integration.md`).

## Optional third arm (exists today, tools-not-chat)

Register team-olai tools into claw-auth's `mcpgateway` (`POST /claw/api/v1/gateway/registry/register`, S2S-key-gated) so Spaces' own Ask-AI agents can query a team's board. No public execution endpoint — internal invocation only.

## Open rulings (only matter at phase 2)

1. **Authority**: who in the channel counts as "the human" for gates (merge-human, deferrals)? Simplest: anyone with channel write — the team collectively — attributed per message.
2. **Channel granularity**: one channel per team-orchestrator, or per lane? (The doorbell digest already compresses well; per-lane rooms multiply.)
3. **Org memory**: do transcripts/lane events also flow into the Spaces context store as a connector (searchable org-wide)? Separable; phase 3.
