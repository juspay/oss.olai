# The deterministic ban — nothing kills the production server by accident

*Brainstorming, 2026-08-29. Born from an incident: the production `olai.service` took external SIGTERMs at 08:56:38 and 09:31:20 (no `systemctl` stop job in the journal — someone on the machine signalled the process directly). The sender is unrecorded: same-user signals leave no trace on a stock Linux. Suspect pool: agent lanes that start and kill test olai servers with ad-hoc cleanup commands.*

## The problem, precisely

Everything on this machine runs as one user. Any process that user owns — including every agent lane — can signal any other, including the production server. A lane running `pkill -f olai` to clean up its own test server matches production too. Agent discipline reduces the odds; it cannot make the odds zero. "Deterministic" means the protection lives in the victim, not in the manners of the attackers.

## Solution 1 — the server refuses unattributed TERMs (an olai PR)

A default SIGTERM handler kills the process without telling you who asked. POSIX offers better: install the handler with `sigaction(2)` and the `SA_SIGINFO` flag, and the handler receives a `siginfo_t` carrying **`si_pid` and `si_uid` — the sender's process id and user id**.

    struct sigaction sa = {0};
    sa.sa_flags = SA_SIGINFO;
    sa.sa_sigaction = on_term;          // void on_term(int, siginfo_t *info, void *)
    sigaction(SIGTERM, &sa, NULL);
    // info->si_pid, info->si_uid — then read /proc/<si_pid>/cmdline before it exits

Bun has no built-in siginfo, so this lands via `bun:ffi` (a ~30-line binding to `sigaction`, or a tiny preloaded native shim).

The policy in the handler:

- Sender is the **user systemd manager** (the service's own parent — how `systemctl stop`/`restart` delivers TERM) → honor it: orderly shutdown as today.
- **Anyone else** → log one loud line — `refused SIGTERM from pid 12345 uid 1000 (bun evidence.ts …)` — and carry on serving.

A stray `pkill` can no longer kill production, and every attempt is *named* in the journal. Two footnotes to keep the promise honest:

- systemd's stop sequence escalates: TERM, then after `TimeoutStopSec`, KILL. Honoring the manager's TERM keeps ordinary stops fast; the escalation path is untouched.
- `kolu watch`-style tooling and the dev loop never signal the production unit, so nothing legitimate is broken.

### Can root kill it anyway?

Yes — and it is worth being exact about how:

- **root's SIGTERM**: still *catchable*. Under the policy above it would be logged and refused like anyone else's (signal disposition belongs to the receiving process; root can send any signal but cannot force a process to act on a catchable one). If we ever want "root's TERM is honored," that's one `si_uid == 0` clause — but refusing it and logging is the safer default, since nothing legitimate sends root TERMs here.
- **root's SIGKILL (`kill -9`)**: uncatchable, unblockable, by design — `SIGKILL` and `SIGSTOP` cannot be handled by any process, ever ([signal(7)](https://man7.org/linux/man-pages/man7/signal.7.html)). Root (or the same user!) can always `kill -9` production, and the process gets no chance to log the sender. So solution 1 is a deterministic ban on **TERM-class accidents** (which is what agent cleanups send); it is not armor against KILL. Attribution for KILL exists only outside the process — which is solution 3's job.
- The kernel's OOM killer also uses SIGKILL; same story.

## Solution 2 — house law: group-scoped kills only (process-file amendment)

Lane agents never use `pkill`/`killall` with `-f` patterns. Cleanup is scoped to what the lane itself spawned:

    setsid bun serve.ts &            # child in its own process group / session
    pid=$!
    ...
    kill -- -"$pid"                  # the NEGATIVE pid: signal that GROUP only

This is already the discipline saatchi (`set -m` + group kill + `pkill -P` parent-scoped fallback) and the e2e reaper encode; the amendment writes it into the briefs' stop-lines so ad-hoc cleanups follow it too. Not deterministic by itself — it is the hygiene layer under solution 1's guarantee.

## Solution 3 — auditd: the machine remembers every kill (nixos-config) — **DEPLOYED 2026-08-29**

*The human landed this in nixos-config and deployed it to naiveintent the same morning; `auditd` verified active. The next unexplained signal is one `ausearch` away.*

The Linux audit subsystem can record **every kill-family syscall** with the sender's pid, uid, command, and the target — root included, SIGKILL included, because the record is written by the kernel at syscall entry, not by the dying process.

### The pedagogy, briefly

`auditd` is a kernel-backed logging daemon: you give the kernel *rules* (via `auditctl` or `/etc/audit/audit.rules`), the kernel emits *events* to the daemon, and `ausearch`/`aureport` query the log. Rules on syscalls take the shape "on every exit from syscall S, if fields match, log with key K."

The kill family is four syscalls: `kill`, `tkill`, `tgkill`, and `pidfd_send_signal`. A focused rule set:

    # every TERM/KILL any process sends (a0=target-pid lives in the record; a1=signal)
    -a always,exit -F arch=b64 -S kill -F a1=15 -k sigterm-audit
    -a always,exit -F arch=b64 -S kill -F a1=9  -k sigkill-audit
    -a always,exit -F arch=b64 -S tkill,tgkill,pidfd_send_signal -k sig-audit

Querying after an incident:

    ausearch -k sigterm-audit --start today
    # → one record per send: the SENDER's pid/uid/comm/exe, the syscall args
    #   (a0 = target pid in hex), timestamp to the millisecond.

Had this been running today, "who TERMed olai at 09:31:20" would be one `ausearch` away, with the sender's full command line — no cooperation from anyone required.

### On NixOS

    security.auditd.enable = true;
    security.audit.enable = true;
    security.audit.rules = [
      "-a always,exit -F arch=b64 -S kill -F a1=15 -k sigterm-audit"
      "-a always,exit -F arch=b64 -S kill -F a1=9  -k sigkill-audit"
    ];

Cost: negligible for signal-only rules (signals are rare syscalls; this is not file-access auditing). The log lives under `/var/log/audit/`.

### Reading

- [signal(7)](https://man7.org/linux/man-pages/man7/signal.7.html) — dispositions; why KILL/STOP are uncatchable
- [sigaction(2)](https://man7.org/linux/man-pages/man2/sigaction.2.html) — `SA_SIGINFO`, `si_pid`/`si_uid`
- [audit.rules(7)](https://man7.org/linux/man-pages/man7/audit.rules.7.html) and [auditctl(8)](https://man7.org/linux/man-pages/man8/auditctl.8.html)
- [RHEL security guide — auditing the system](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/security_hardening/auditing-the-system_security-hardening) — the canonical worked examples
- [NixOS options: security.audit](https://search.nixos.org/options?query=security.audit)

## How the three compose

| Layer | Stops the accident? | Names the sender? | Covers root/KILL? | Lives in |
|---|---|---|---|---|
| 1 — refuse + attribute | **yes** (TERM-class) | yes (TERM-class) | no | olai (the server) |
| 2 — group-scoped kills | reduces odds | no | no | process files (briefs) |
| 3 — auditd | no | **yes, always** | **yes** | nixos-config |

1 is the ban, 2 is the hygiene, 3 is the black box recorder. All three are small; none conflicts with the others.
