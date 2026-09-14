# Plan: issue #113 — `attach` appears hung with a large catalog

Resolves <https://github.com/juspay/odu/issues/113> in ONE pull request, opened
as a **draft** by the implementor and left for a human to review and merge.
The implementor never merges it.

The issue names two compounding problems and this plan fixes both, plus the
third thing the report implies: a client that can wait forever with nothing on
screen. The organising idea is **algorithmic**, not incremental: every path on
the report is `O(n · J)` or `O(n)` round trips where `n` is the catalog size
and `J` a journal's length, and each is brought to `O(1)` round trips or
`O(n)` `stat`s with no parsing.

---

## 1. Where the time goes today (cost model)

Let `n` = runs in the catalog (710 on the reporting host), `J` = lines in a
run's journal, `L` = runs currently live, `T` = `REFRESH_MS` = 250 ms.

### 1.1 Daemon refresh — `packages/service/src/registry.ts:438`

`refresh()` runs synchronously on the daemon's event loop every `T`. It calls
`listRuns()` (`packages/run-history/src/store.ts:790`) and then **uses only
`listed.runId`** — every other field it paid for is discarded. Per run, per
tick, for an UNCHANGED settled run:

| work | where | cost |
| --- | --- | --- |
| `readdirSync(catalog)` + sort | `listRuns` | `O(n log n)` once per tick |
| manifest read + `JSON.parse` + Schema decode | `listRuns` → `readManifest` | 1 file |
| `currentOwner`: `owner.json` read + `readdirSync(dir)` + claim reads | `listRuns` | 1 file + 1 readdir |
| verdict read + parse + decode | `listRuns` → `readVerdict` | 1 file |
| expiry read (usually ENOENT) | `listRuns` → `readExpiry` | 1 syscall |
| **journal read + parse + decode + sort of every line** | `listRuns` → `resumedSinceFinal` → `readJournal` | **`O(J)`** |
| `currentOwner` AGAIN | `registry` → `ownerProvablyAlive` | 1 file + 1 readdir |
| 3 × `statSync` | `registry` → `fingerprint` | 3 syscalls |
| fingerprint compare → **skip** | `registry` | — |

So the fingerprint that exists to skip unchanged runs is evaluated **after** the
run has already been fully parsed. Per tick: `O(Σ J_i)` JSON decode work plus
~8 syscalls per run, all synchronous. Measured on the host: 700–1100 ms per
warm tick against a 250 ms interval, i.e. the loop is busy ≈100 % of the time
and `setInterval` ticks queue back-to-back. Every RPC waits behind a tick.

### 1.2 Client board read — `packages/cli/src/serviceFace.ts:390`

`readRows()` = 1 round trip for `runs.keys` + **`n` serial round trips** for
`runs.get({key})`, each awaited to its first frame. Each round trip lands on a
starved loop, so latency ≈ one refresh (~1 s). Total ≈ `n` seconds: 11 min at
710 rows, before the first byte reaches the terminal. Callers:

- `statusViaService` / `attachViaService` → `openHere` → `readRows`
  (`serviceStatus.ts:122`), then filters to ONE row client-side.
- `resolveRunAddress` for `latest` and `<sha7>#<seq>` (`serviceFace.ts:442`) —
  used by `wait`, `rerun`, `cancel`, `history show`.
- `listViaService` (`serviceCommands.ts:614`) — `history list`.

No first-response deadline anywhere on this path: `firstFrame` is a bare
`Stream.runHead` (`packages/execution/src/common/effectEdge.ts:136`), and
nothing is written to the terminal while waiting.

### 1.3 Targets

| path | before | after |
| --- | --- | --- |
| warm refresh tick, unchanged catalog | `O(n·J)` parse + ~8 syscalls/run | **4 `stat`/run, zero parses, zero reads** |
| refresh of a run whose files moved | `O(J)` for that run | unchanged (`O(J)` for that run only) |
| cold refresh (daemon start) | `O(n·J)` | unchanged, but before `ready` only |
| daemon loop occupancy at n=710 | ≈100 % | ticks of a few ms; idle gap ≥ `T` guaranteed |
| `odu status` / `attach` / `--run latest` | `1 + n` serial round trips | **1** round trip + 1 nodes frame, both deadline-bounded |
| `<sha7>#<seq>` | `1 + n` | 1 |
| `history list` (this checkout) / `--all` | `1 + n` | 1 (payload = the answer's own size) |
| feedback while waiting | none | one stderr line after 250 ms; failure reported by 10 s |

---

## 2. Changes, in dependency order

The arrow is `cli → service-client → service → run-history`; edit bottom-up so
each layer compiles against the one below it.

### 2.1 `@odu/run-history` — cheap discovery (`store.ts`)

Add `listRunIds(opts): string[]` — `readdirSync(catalog)`, `filter(isRunId)`,
newest-first string sort. **Reads no file.** `listRuns` stays for its other
callers (`probeCheckout`, `resolveRunRef`, `latestRun`, the legacy query) and
keeps its shape.

### 2.2 `@odu/service` — fingerprint before parse (`registry.ts`)

1. `refresh()` iterates `listRunIds` instead of `listRuns`. Nothing the
   registry used is lost: `project()` already re-derives `resumed`, the
   verdict and the owner from its own single journal read.
2. Make the fingerprint **pure stat**: `stat` of `events`, `verdict.json`,
   `owner.json` **and the run directory itself**. The directory mtime is what
   makes a takeover claim file (`owner.<epoch>.claim`, created exclusively,
   never rewritten) move the fingerprint without a `readdir` per tick.
3. Cache the decoded `Owner` record on `Entry` (written by `project()`). On a
   tick where the four stats are unchanged, liveness is recomputed as
   `ownershipProvablyLost(cachedOwner, now)` — arithmetic plus, only once the
   grace has passed, one `kill(pid, 0)`. This keeps the existing
   "owner_lost with nothing on disk moving" behaviour (both registry tests for
   it must keep passing unchanged) while removing the per-tick `currentOwner`
   reads. Equivalence argument, to go in the module comment: `currentOwner` is
   a function of `owner.json` and the claim-file set; the first is covered by
   its own stat, the second by the directory stat, so a run whose four stats
   are unchanged has the same `currentOwner` it had when it was projected.
4. Keep `RegistryDelta` as is. Add `durationMs` to it only if the scheduler
   below needs it for its warning; otherwise measure in the scheduler.

### 2.3 `@odu/service` — the poller cannot pile up (`service.ts:602`)

Replace `setInterval(refresh, T)` with a self-rescheduling `setTimeout` armed
**after** a refresh completes, so consecutive ticks are separated by at least
`T` of idle loop regardless of how long a tick took. Keep `unref()`. Add a
rate-limited (once per minute) warning through the daemon's existing logger
when a tick exceeds `T`, naming the tick duration and `n` — the observability
this report lacked. Extract the scheduler as a small pure helper
(`everyAfter(fn, ms, timers)`) so it is unit-testable with injected timers.

No time-budgeted/partial refresh: with 2.2 in place a 10 000-run warm tick is
~40 000 `stat`s, tens of ms. Note this reasoning in the header comment so the
next person does not add chunking without a measurement.

### 2.4 `@odu/service-client` — one declared query (`surface.ts`, `verbs.ts`)

Add procedure **`run.list`** (MCP/CLI name `run_list`, `mutates: false`):

```ts
input:  { checkout?: string, sha?: string, seq?: int, limit?: int }
output: { rows: RunRow[], total: int }   // newest first
error:  ServiceRefused — checkout_refused (relative checkout), bad_input (seq without sha)
```

Filters are ANDed and served from the registry's in-memory projection: a
single `O(n)` scan over `order`, `rows` cut at `limit`, `total` the full match
count. `checkout` is an explicit absolute path — the same rule `run.start`,
`pipeline.read` and `venue.hold` already keep, so this does not make the
service "know where the caller is standing"; the client still resolves "here"
with `git rev-parse` and sends it. Run through `notAbsolute("run.list", …)`.

Bump `SERVICE_CONTRACT_VERSION` to `"1.4"` (additive minor). Consequence: a new
client against an older daemon is refused with the existing
"Run `odu web --upgrade`" message — which is the right outcome on the reporting
host, whose 5-day-old daemon is the CPU hog.

Add `run.list` to `ODU_SERVICE_EXPOSE` and one sentence to
`ODU_SERVICE_MCP_INSTRUCTIONS` ("`run_list` filters the board by checkout,
commit and seq without fetching every row").

### 2.5 `@odu/service` — implement it (`service.ts` procedures)

`run.list` → `registry.select({checkout, sha, seq, limit})`. Add `select` to
`RunRegistry`. `sha` matches by lower-cased prefix (≥ 7 hex, else `bad_input`),
`seq` exact. Freshness: the projection lags the catalog by at most one tick,
and `run.start` already calls `refresh()` before answering, so a run the caller
just started is visible to its own next `run.list`.

### 2.6 `@odu/cli` — one round trip, bounded, with feedback

- **Delete `readRows`** and the structural `keys`/`get` cast with it. Every
  former caller goes through `run.list`:
  - `openHere` → `run.list({checkout, limit: 1})`; `currentRun` keeps its
    `NOTHING_TO_SHOW` policy on the returned row (it is a face decision).
  - `resolveRunAddress`: `latest` → `run.list({checkout, limit: 1})`;
    `<sha7>#<seq>` → `run.list({sha, seq, limit: 1})` (global, newest wins,
    matching `resolveRunRef`). Run ids still pass through untouched.
  - `listViaService`: `run.list({checkout?, limit?})`; `--all` omits
    `checkout`. Client-side sort/filter/slice go away.
- **Bounded first response.** Add an optional `{ deadlineMs }` to
  `effectEdge.firstFrame` (this is the one Effect boundary module; do not add a
  second `Effect.run*` in `cli`). On expiry it rejects with a named error, not
  `undefined`, so "empty stream" and "no answer" stay different facts. The CLI
  wraps the `run.list` call and the first nodes frame with 10 s; on expiry it
  prints one sentence naming the origin and exits 3 (the documented "nothing
  serving" code; `reportLost` is the model).
- **Visible waiting.** A helper `withFeedback(promise, line)` in
  `serviceFace.ts`: when `stderr` is a TTY and the promise is still pending
  after 250 ms, write one line (`odu: waiting for the service at <origin>…`),
  and nothing otherwise. Applied to the dial, the `run.list` call and the first
  nodes frame. Never on stdout; never in `-o json`'s stdout stream.
- **Untouched:** `watchNodes` (24 h re-subscribe loop), `nodesStream`'s fence,
  `dispose()` in `finally`, the live matrix, the exit tables.
- Update `serviceStatus.ts`'s module header (the "resolved on this side" prose
  now describes an explicit `checkout` on the query, not a client-side filter)
  and remove the `readRows` doc block whose cast history no longer applies.

### 2.7 Docs sync (required by `.claude/rules/surface-docs-sync.md`)

One-line entries for `run_list` in **all three**, same commit:
`README.md` verb table, `website/src/content/docs.md`, and
`.apm/skills/odu/SKILL.md` ("Beyond one run"), then `just apm`. Never edit
`.claude/skills/**` directly.

---

## 3. Tests

Unit: `bun run test:unit`. E2E: `bun run test:e2e-cli`. Typecheck:
`bun run typecheck`. All must be green before the draft is opened.

### 3.1 `packages/run-history/src/store.test.ts`
- `listRunIds` is newest-first, ignores non-id entries, and lists a run
  directory that contains **no files at all** (proof it opens nothing).

### 3.2 `packages/service/src/registry.test.ts`
- **Warm refresh reads no journal (requirement-shaped):** 200 settled runs
  across 3 checkouts, each with a ~1 500-line journal written in one
  `writeFileSync` by a new fixture helper (`writeBulkJournal`); cold refresh
  projects all 200; the next refresh has `upserted: []` **and completes in
  under `REFRESH_MS`**. A cold parse of 300 000 lines is seconds, so the bound
  discriminates by an order of magnitude and cannot flake in the passing
  direction.
- A takeover claim file appearing with `owner.json` untouched re-projects the
  run (pins the directory-stat rule).
- Existing owner-lost-by-time tests unchanged (they now exercise the cached
  owner path).
- Resumed finalized run: `finalizeRun`, then a new `attempt_started` → row is
  `running`, `outcome` null; finalize again → `settled`.
- Expiry: `expireRun` on a settled run → row `expired` on the next tick.
- Manifest-less directory: skipped, never published, not counted in `select`.
- `select`: checkout filter, sha-prefix + seq, limit and `total`, newest-first
  order, absent filters return everything.

### 3.3 `packages/service/src/service.test.ts` (or `list.test.ts`)
- `run.list` through the runtime: refusals (`checkout_refused`, `bad_input`),
  a run started via `run.start` is visible to an immediate `run.list`.
- Scheduler helper: with injected timers, the next arm happens after the
  callback returns and two callbacks never overlap.

### 3.4 `packages/cli` (stand-in client, like `liveFromService.test.ts`)
- `status`/`attach` resolve through `run.list`; cases: no run (exit 0, the
  "no run in flight" line), newest `expired` → none, newest `owner_lost` →
  shown and exit 3, settled red → exit 1, first nodes frame past the deadline →
  exit 3 with the bounded message, feedback line appears after the threshold
  (threshold injected) and not in JSON mode.
- `resolveRunAddress`: `latest`, `<sha7>#<seq>`, unknown → exit 4 unchanged.

The `serviceFace.ts` header records that stand-in tests missed two cast bugs,
so the wire itself is covered below.

### 3.5 `tests/e2e/catalog-scale.e2e.test.ts` (black-box, nix-built binary)
- Private world via `webHarness.privateWorld`; run the `pass` fixture once to
  obtain one real settled run directory; **clone it 1 000 times** under
  `<ODU_STATE_DIR>/runs` across 3 fake `repoRoot`s, rewriting the run id
  (`<8+ base36>-<4+ base36>`) inside `manifest.json`, `events` and
  `verdict.json` by string replacement, then start the daemon.
- Assertions, each with a generous CI bound (10 s) and a recorded wall time in
  the test output:
  - `odu status` in a fresh checkout with no run answers "no run in flight";
  - `odu status -o json` in the seeded checkout names its newest run;
  - `odu history list --all -o json` returns ≥ 1 000 rows in one call;
  - `odu wait --run latest` resolves;
  - `odu surface service` (the identity cell) answers in < 1 s while the
    catalog is large — the "RPC stays responsive during refresh" check.
- `odu attach` on a live `sleep` fixture run with the seeded catalog shows the
  matrix before the run settles (drive from a second process as `cancel.e2e`
  does).

---

## 4. Evidence (per `.agency/do.md`)

`tests/evidence/big-catalog-demo.sh`: seed 1 000 cloned runs into a private
`ODU_STATE_DIR`, `odu web` there, then `time odu status` in a no-run checkout,
`odu history list --all | wc -l`, and `odu attach` on a quick run. Record with
asciinema, render with agg, upload to the `evidence-assets` release, embed in a
`## Evidence` PR comment alongside a `console` block showing warm-tick timing
from the daemon log before/after.

---

## 5. PR mechanics

- Branch `issue-113` (this worktree). Commits in the order of §2 so each is
  reviewable; the contract bump and the three docs edits in one commit.
- `gh pr create --draft` titled for the behaviour (e.g. "attach/status answer
  in one round trip; catalog refresh stats instead of parsing"), body per the
  `forge-pr` skill, `Closes #113`. **Do not merge.**
- CI is the repo's own: `just ci` recipes (`typecheck unit fmt nix e2e-cli
  e2e-web bun-nix-fresh`). Note in the PR that the contract bump means
  existing daemons need `odu web --upgrade`.

---

## 6. Non-goals and follow-ups (state in the PR, do not do)

- `probeCheckout` (`packages/cli/src/webPorts.ts:86`) still calls `listRuns`
  per `run.start`: `O(n)` manifest parses on the daemon loop per start, not per
  tick. Follow-up: serve it from the registry, or make `listRuns`'s `resumed`
  lazy.
- The browser board and `odu surface keys/get/deltas` are untouched; the
  `deltas` opening frame already carries the whole board in one message.
- No change to ownership, grace or heartbeat semantics.

## 7. Decisions left to the reviewer

1. Verb name `run.list` vs `run.find`/`run.resolve` — plan recommends `list`
   (it is a filtered listing, and `history list --all` uses it unfiltered).
2. First-response deadline 10 s and feedback threshold 250 ms — plan
   recommends both as constants in `serviceFace.ts`, not flags.
