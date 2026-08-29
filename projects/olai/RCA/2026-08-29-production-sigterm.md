# RCA: the production server killed twice by lane cleanups (2026-08-29)

## What happened

The production `olai.service` (serving this vault) received external SIGTERMs at **08:56:38** and **09:31:20** and was revived by systemd's restart policy within a second each time. No data lost; each outage also killed the orchestrator's chat session and its fleet watcher. The human did not deploy; the journal showed no `systemctl` stop job for either (unlike the legitimate 08:34 deploy, which logged one).

## Root cause, with the evidence chain

Two agent lanes, cleaning up their own test servers, each ran a broad process kill matching by a **shared source-path substring** that appears in every olai server's command line — production included:

- 08:56 — the pi-acp lane ran `pkill -f "main.ts web"` (admitted in its audit; it believed the pattern harmless).
- 09:31 — the feed-mutes-footer lane ran `pkill -f "packages/server/src/main.ts web"` (confessed in its audit; it noticed the foreign processes moments after firing).

The one alibi offered — "the Vault server matches the same substring and survived, so my pkill can't have fired" — was voided by `ps`: the Vault server runs as uid 1001000, a different user, which `pkill` from the lanes' uid cannot signal. Production was the only same-uid match, and it died both times.

Contributing conditions: everything runs as one user, so any lane can signal production; SIGTERM senders are unrecorded on a stock system (no auditd), so attribution required interrogating the agents rather than reading a log.

## What held

systemd's `Restart=` policy (1-second recovery, twice); the vault's durability (no writes lost); the agents' honest self-audits when confronted.

## The fix (proposed — see brainstorming/deterministic-ban.md for the full design)

1. **Deterministic**: the server catches SIGTERM via `sigaction`+`SA_SIGINFO`, honors it only from the systemd user manager, and logs-and-refuses every other sender with pid/uid/cmdline. A stray pkill then cannot kill production and every attempt is named. (olai PR, awaiting dispatch.)
2. **Hygiene**: house law — no `pkill`/`killall -f` patterns in lanes, ever; cleanup is explicit-pid / process-group / port-scoped only. (Process-file amendment, awaiting approval.)
3. **Forensics**: auditd kill-syscall rules in nixos-config, so the next unexplained signal is one `ausearch` away. — **DEPLOYED by the human, 2026-08-29, same morning; auditd verified active.**

Interim, effective immediately in lane instructions: kills scoped by explicit pid or port only.
