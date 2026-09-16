# Plan: the git row integrates the upstream before it pushes

One PR, opened as a **draft** on this worktree's branch (`git-pull-before-push`) and **never merged by the implementor**. The human approves this plan first, then reviews and merges the PR.

## 1. The rule this PR adds

> A push olai makes first takes in what its upstream already has, by rebasing the unpushed commits onto it, so that a push is never refused as a non-fast-forward for a reason olai could have removed itself.

Everything below is that sentence made concrete, and the fences around it.

## 2. The failure, as the human met it

The human's screenshot (a private vault; it is described here and not quoted, and nothing from it goes into the PR, its tests, or its commit messages) shows a vault served at the repository root with auto-commit and auto-push both on. The quiet window recorded one commit. The push that followed was refused by the remote because the remote held a commit this machine did not have. The loop paused, and the panel showed git's own refusal, whose advice is to fetch and integrate before pushing again.

Nothing was wrong with the vault. The other machine had pushed, which is the ordinary life of one person with two machines. Today's design ([docs/git.md](docs/git.md) "When it stops") calls this a conversation in a terminal. For a person who turned on auto-push precisely so as not to open a terminal, that is the feature refusing to do its job on every second machine, every time the first one moved.

## 3. Decision: rebase, not merge

Olai rebases its unpushed commits onto the upstream. It does not merge, and it does not honour `pull.rebase`.

**Why rebase**

- The audit trail stays one line. `git log --grep '^olai'` is the audit view, and every commit in it keeps its `olai:` subject and its `X-Olai-Writer` trailer through a rebase. A merge would add a commit olai composed, with no writer of its own, on every sync, and two machines auto-pushing would braid the history of a notes directory into a ladder.
- "Never amend" is not violated. That rule (`git.ts`, `commit`) is about commits that may have been pushed. What a rebase rewrites is exactly the set that has not been: the count the pill calls `unpushed`. Content, message, author and trailer survive; only the parent changes.
- A rebase fails in exactly one more way than a merge does, and it is the same way: a content conflict. There is no failure mode a merge would have avoided.

**Why not honour `pull.rebase`**

Olai never runs `git pull`. The shape of olai's own history is olai's decision, made once here, and a person's `pull.rebase=false` was set with their typing in mind, not olai's. Reading it would make what auto-push does depend on a setting nobody set for it.

**No new setting.** `push` keeps its two modes (`@olai/format`'s `PUSH_MODES`, "deliberately not three"). Integrating is part of what pushing means now, on every door: the Push button, the agent's `git_push`, the commit that auto-push follows, and the one push at boot. A knob that turned it off would be a knob for "stop the loop on every second machine", which is the bug.

**What integrating is not.** It is not sync. Olai fetches and rebases only when it is about to push and only when there is something to push. It does not fetch on the sweep, and it does not pull when the branch has nothing to send. "There is no pull and no fetch" in the docs becomes "olai fetches and rebases exactly once per push, and never otherwise".

## 4. The mechanics

Verified by hand against git 2.55 in a scratch repository (a bare remote, a diverged clone, two local `olai:` commits, one file modified but unstaged, one file with a staged edit and a further unstaged edit on top). Every claim below was observed, not recalled.

### 4.1 The recipe

Under the ledger's push verb, once the survey says the branch is `Ready` and has commits to send:

1. **`git fetch`**, bare. It fetches the upstream's remote and moves the remote-tracking refs. Nothing else in olai ever does this, so the `behind` count git can print is meaningless until this runs.
2. **Where the branch stands.** `git rev-parse --symbolic-full-name @{upstream}` and `git rev-list --left-right --count @{upstream}...HEAD`, giving the upstream's full ref, `behind` and `ahead`. Neither touches the index. `behind == 0` skips to step 6.
3. **A linked worktree of olai's own**, inside the git directory beside the index backups `keptIndex` already writes there: `git worktree add --no-checkout --detach <gitDir>/olai-integrate-<pid>-<n> <HEAD>`, then `git reset --hard <HEAD>` inside it. `--no-checkout` plus `reset --hard` is what keeps the project's `post-checkout` hook from running. `git status` in the served tree does not see the worktree (observed), and the served tree's `state` stays `Ready` throughout, because the rebase's `rebase-merge` directory lives under `.git/worktrees/<name>/`, not in the main git directory (observed).
4. **The rebase, in that worktree:** `git -c rebase.updateRefs=false -c rebase.autoStash=false -c rerere.enabled=false rebase --no-verify <upstream ref>`. Config keys rather than flags so an older git ignores them instead of refusing them. `updateRefs` off so no other branch of the person's is moved. `rerere` off so a recorded resolution can never be taken on olai's behalf: a conflict is a conversation, every time. `--no-verify` skips `pre-rebase` for the reason `commit` skips its hooks. Signing is not skipped, for the reason it is not skipped on commit: the rebase honours `commit.gpgsign`, and where a key is missing the commit before it would already have failed.
   - **Conflict:** `git rebase --abort` in the worktree. The served tree was never touched.
5. **Moving the served tree**, under the index gate, uninterruptible, in this order:
   1. `git read-tree -m -u <old HEAD> <rebased HEAD>` in the served tree. A two-tree merge: it updates the index and working tree for every path that differs between the two commits, keeps a hand-staged index entry and an unstaged edit for every path that does not (observed: the staged edit and the unstaged edit on top of it both survived), and refuses before writing anything when a path the upstream changed has local changes (`Entry 'x' not uptodate. Cannot merge.`) or an untracked file would be overwritten (both observed, and in both cases the tree was untouched afterwards).
   2. `git update-ref -m "olai: integrated <upstream>" refs/heads/<branch> <rebased HEAD> <old HEAD>`. The third argument is a compare-and-swap, so a commit typed in a terminal during step 4 makes this refuse rather than being dropped from the branch.
   3. If the compare-and-swap refuses: `git read-tree -m -u <rebased HEAD> <old HEAD>` puts the tree back, and the outcome is a refusal with words.
6. **`git push`**, bare, exactly as today.
7. The worktree is removed (`git worktree remove --force`) on **every** exit of step 3 onward: success, conflict, refusal, defect, interrupt.

### 4.2 What was tried and rejected

- **`git rebase` in the served tree itself.** Refuses on any unstaged tracked change, so under manual mode with an unticked file it would never integrate; and a conflict would put markers into files the store is watching, for as long as git takes to be told to abort. A crash mid-rebase would leave the person's vault mid-rebase.
- **`--autostash`.** When the stash cannot be reapplied, git resets the tree hard and leaves the edits in the stash. The file on disk then no longer says what the person typed a second ago. Never.
- **`git reset --keep <rebased>`** for step 5. It preserves the working tree but resets the index to the target for a path with a staged edit, which was observed to discard a hand-staged entry. That breaks "a selection is never git's index" ([docs/git.md](docs/git.md)), so `read-tree -m -u` plus `update-ref` replaces it.
- **`git checkout -B <branch> <rebased>`** for step 5. Preserves staging (observed) and is one command, but runs `post-checkout` and has no compare-and-swap on the ref.
- **`git replay`** (git 2.44+) is the exact primitive: a rebase with no worktree, emitting `update-ref` lines. It is still marked experimental in git's own manual, and the server runs whatever `git` is on its PATH. Noted as what steps 3 and 4 collapse into once it is not.
- **Reimplementing rebase with `merge-tree --write-tree` and `commit-tree`.** No worktree, but it would be olai's own rebase, with author, message, trailer, empty-commit and signing behaviour to get right by hand. The plumbing file's rule is that git's verbs do the work.
- **Push first, and integrate only on a non-fast-forward refusal.** One round trip fewer in the common case, but it means classifying the refusal by matching git's prose, and it is a retry: the thing the docs refuse. Fetch-first is git's own advice and the same path every time.

## 5. Policy: what each outcome does

All of this lives in `packages/plugins/git/src/ledger/pending.ts`'s `push`. The plumbing (`git.ts`) decides nothing.

| step | outcome | `PushResult` | `pushSaid` | pauses the loop (under `commit: auto`)? |
| --- | --- | --- | --- | --- |
| survey | `ahead == 0` | `NothingToPush` | cleared by the survey | no |
| fetch | refused | `Failed`, git's words | git's words | yes, as a refused push today |
| standing | no upstream | falls through to `git push` as today, whose refusal names it | git's words | yes, as today |
| standing | `behind == 0` | push as today | | |
| integrate | `Integrated` | continues to push | | |
| integrate | `Overlapped` (uncommitted edits, or an untracked file, in a path the upstream changed) | `Failed`, olai's sentence naming the path plus git's words | set | **no** |
| integrate | `Conflicted` | `Failed`, olai's sentence naming the conflicting paths plus git's words | set | yes |
| integrate | `Refused` (rebase could not run, the branch moved under it, a hang) | `Failed`, git's words | set | yes |
| push | `Pushed` | `Pushed { upstream, commits, integrated }` | cleared | no |
| push | refused (the remote moved again in the window, authentication, a hook) | `Failed`, git's words | set | yes, as today |

**Why `Overlapped` does not pause.** The next commit is the cure. Under auto, the overlapping file is dirty, so the survey after this push re-arms the window, the window commits it, and the push that follows that commit integrates cleanly. Under manual, there is no loop to pause, and the Push button's refusal tells the person which file to commit first. A pause here would make the person press Resume to lift a stop the loop would have walked out of on its own.

**Why a conflict pauses.** It is the case today's design was built for: a person has to look. The commit stands, the served tree is exactly as it was, no rebase is in progress anywhere the person can see, and the words on the pill name the files and say what to do (`git pull --rebase` in a terminal, resolve, then Resume).

**One push in flight per directory.** `push` takes a permit owned by the `Committing` instance. Without it, a Push button pressed during auto-push's own push would run a second integrate whose compare-and-swap refuses against the first's ref move, and the loop would pause over olai racing itself. The second caller waits, surveys again, and finds nothing to push. `whyWaiting` and the survey do not take this permit.

**Boot (`catchUp`)** takes the same path. A machine that was off for a week comes up, takes in what the other machine pushed, pushes its own, and the pill reads clean. A conflict at boot re-earns the words exactly as a refusal does today.

## 6. What stays true

- The served working tree changes in exactly one way: the two-tree update of step 5, which either applies whole or refuses whole. No conflict marker and no rebase state ever appears in it.
- A hand-staged index entry for a path the upstream did not change is bit-identical afterwards. An unstaged edit is untouched either way.
- Nothing is forced, nothing is amended, nothing is retried, and nothing clears a pause but Resume or a restart.
- `Blocked` repositories (mid-merge, mid-rebase, detached) are refused before any of this, as today.
- Git's words are still on the pill, on its `aria-label`, and in the panel, whole. Where olai adds a sentence of its own, git's words follow it.
- The panel's last-commit row shows a different short sha after a rebase, because the commit is a different object. The docs say so.

## 7. Cordis: owners, lifetimes, boundaries

- **No new service, plugin, or config.** The ledger door (`Ledger`) keeps its shape; `Ops`, `Vault`, `Surfaces` are used as today. The `git` row's `Config` is unchanged.
- **The temporary worktree is a resource with one owner.** It is acquired and released with `Effect.acquireUseRelease` inside the plumbing's `integrate`, on the model of `keptIndex`: every exit, including interruption when the row is switched off or the server stops, removes it. Step 5 is one `Effect.uninterruptible` region so a stop cannot land between `read-tree` and `update-ref`.
- **The index gate is reused, not duplicated.** `integrate` holds `holdIndex(gitDir, …)` for the whole of steps 3 to 5, so no olai commit can land between the rebase reading `HEAD` and the ref moving. `fetch`, `standing` and `push` stay off the gate, as `push` is today, so `whyWaiting` never queues behind the network.
- **The push permit** is a `Semaphore` made in `make`, owned by the `Committing` value, which is one per served directory and goes with it.
- **Crash residue has an owner too.** At the start of every `integrate`, worktrees named `olai-integrate-<pid>-<n>` whose `pid` is not alive are removed with `git worktree remove --force` and `git worktree prune`. A live pid that is not this process is left alone.
- **Static contracts change by import.** `PushResult.Pushed` gains `integrated: Schema.Int` in `@olai/format`; the browser and the tool read it as a type.
- **Shutdown cost, named.** A stop arriving inside an integrate waits out at most the uninterruptible step 5 (three subprocesses at `BUDGET`), on top of the commit's own three. The rebase itself is interruptible and its abort is the worktree's removal.

## 8. Code changes, file by file

### `packages/plugins/git/src/git/git.ts` (plumbing; decides nothing)

- `Repo` gains three verbs:
  - `fetch: Effect<Fetched>` with `Fetched = { _tag: "Fetched" } | { _tag: "Refused"; said }`.
  - `standing: Effect<Standing | null>` with `Standing = { upstream: string /* refs/remotes/origin/main */; name: string /* origin/main */; ahead: number; behind: number }`. `null` when there is no upstream. Not index-gated.
  - `integrate: (onto: Standing) => Effect<Integrated>` with `Integrated = { _tag: "Integrated"; from: sha; to: sha; taken: number } | { _tag: "Overlapped"; said } | { _tag: "Conflicted"; said } | { _tag: "Refused"; said }`. `taken` is `behind` as counted before the rebase. Index-gated for its whole span; the served-tree update is uninterruptible.
- `open` wires them. `push`, `commit`, `dirty`, `state`, `head`, `show`, `last` are untouched.
- The file header's list of questions and verbs, and its index-gate paragraph, are updated in the same style.
- The `Upstream` type and `tracking()` are unchanged; `behind` is deliberately still not read off `git status`, because that number is stale until a fetch, and the fetch happens on the push path only.

### `packages/plugins/git/src/ledger/pending.ts` (policy)

- `push` becomes: survey → `NothingToPush` / `Blocked` as today → permit → `fetch` → `standing` → `integrate` when `behind > 0` → `push` → `standing` again for the honest `ahead` in `Pushed`.
- `pushed(said, stops)` replaces `pushed(said)`: `Overlapped` sets `pushSaid` without calling `stopBy`. Every other refusal calls it as today.
- Olai's own sentences for `Overlapped` and `Conflicted`, composed here beside `whyOf`, each followed by git's words. They name the paths and the one gesture that helps.
- `catchUp` is unchanged in code; its comment loses "never a pull".
- The header comments on `push` ("There is no pull…") and the `Settled.pushSaid` doc ("what git said when it last refused a push") are rewritten: `pushSaid` is now "why the last push did not go, in git's words and olai's".

### `packages/format/src/committing.ts`

- `PushResult`'s `Pushed` arm gains `integrated: Schema.Int` (commits taken in from the upstream before pushing, `0` when none).
- `GitState.pushSaid`'s doc comment: a refusal of the push, the fetch, or the integration before it.

### `packages/plugins/git/src/tools.ts`

- `git_push`'s description: "Fetches the upstream, rebases what is unpushed onto it, then pushes. Never a force. A conflict is refused with the files named; resolve it in a terminal and report what git said rather than retrying."

### `packages/plugins/git/src/browser/…`

- No new chrome. The pill's "the last push was refused" line and the panel's refused-push text already draw `pushSaid`. If the implementor finds the `Pushed` result rendered anywhere, `integrated` is shown as "· took in N" and nowhere else.

## 9. Tests

The rule from AGENTS.md applies: a green existing suite is not proof. Every row in the table in §5 gets a test at the level that owns it.

### `packages/plugins/git/src/git/git.test.ts` (plumbing, real git)

- `fetch` moves the remote-tracking ref; a remote that cannot be reached is `Refused` with words.
- `standing` reads `ahead`/`behind`/`upstream` and is `null` with no upstream.
- `integrate` on a diverged, clean tree: `Integrated`, the upstream's file is on disk, `HEAD` is the rebased sha, `git worktree list` has one entry afterwards, no `olai-integrate-*` directory remains, and the main git directory holds no `rebase-merge`.
- `integrate` preserves a hand-staged index entry and an unstaged edit for a path the upstream did not change, bit-identical. This is the test `reset --keep` would have failed.
- `integrate` with an uncommitted edit in a path the upstream changed: `Overlapped`, nothing moved, `HEAD` unchanged. Same with an untracked file the upstream added.
- `integrate` on a conflict: `Conflicted` naming the path, `HEAD` unchanged, served tree clean of markers, `state` still `Ready`, no worktree left.
- `integrate` when a commit lands on the branch during the rebase (the test moves the branch from a second process between `worktree add` and `update-ref` using a `pre-rebase`-free hook or a paused fake): `Refused`, tree put back. If this proves unreachable from a test, it is documented as a residue in §11 and the compare-and-swap is still exercised by calling the tree-update step directly with a stale `old`.
- Interrupting `integrate` mid-rebase removes the worktree (fork, interrupt, `git worktree list`).
- A stale `olai-integrate-<deadpid>-1` worktree left by a "previous process" is removed at the start of the next `integrate`.
- The served tree's survey never lists anything from the temporary worktree.

### `packages/plugins/git/src/ledger/pending.test.ts` (policy)

- Rewrite "a refusal surfaces verbatim" and "a push git refuses is remembered, drawn, and stops the loop" to use a **conflicting** remote change (the other clone edits the same file this side commits), since a plain divergence now succeeds.
- New: a diverged push integrates and lands, `unpushed` goes to `0`, `Pushed.integrated == 1`, the other clone's file is on disk, `settlements` fired.
- New: `Overlapped` sets `pushSaid`, does **not** pause under auto, and the next window commit clears it (the loop, forked, commits the overlapping file, integrates and pushes; `pushSaid` reads `null`).
- New: a conflict pauses with a sentence naming the file, the commit stands, the tree is clean of markers, `Resume` then a later commit re-attempts and, once the test resolves the conflict in the "terminal" clone, lands.
- New: a fetch that fails (remote URL pointed at a directory that does not exist) is a refusal with words and a pause.
- New: two concurrent `push` calls make one integration; the second answers `NothingToPush`.
- `catchUp` on a diverged repository integrates and pushes; on a conflicting one, re-earns the words.

### `packages/server/src/headless.test.ts` (boot, real child process)

- "a boot under push: auto re-earns git's words about a branch it cannot send" switches its `diverged` fixture to a conflicting edit of the same file, and keeps its assertions that nothing was forced.
- New: a boot under `push: auto` on a diverged but clean repository takes the other machine's commit in and pushes its own; the bare remote's tip is `olai: earlier`, and the other machine's file is in the served root.

### E2E (`packages/plugins/git/e2e`, `packages/tests/support/world.ts`)

- `world.ts`: `advanceRemote(subject, change?)` takes an optional `{ file, content }` so the other clone can make a real file change. Two new steps in `commit_steps.ts`:
  - `Given somebody else has pushed {string} to the remote` (a file with fixed content).
  - `Given somebody else has pushed a conflicting edit to {string}` (writes a line the scenario's own rewrite of that file will not contain).
  - `Then the served directory has {string}` (the file landed on disk through the integration).
- `committing.feature`:
  - "A branch somebody else has moved stops the loop, and says so" becomes **"A branch somebody else has moved is taken in, and the push lands"**: divergence with a real file, the flurry records itself, the pill says `0 unpushed`, the remote's tip is olai's commit, the served directory has the other side's file, the pill says auto-commit is `armed`, no page errors.
  - New: **"A conflict with somebody else's edit stops the loop, names the file, and Resume starts it again"**: the conflicting step, the flurry records itself, the pill says `paused`, explains `CONFLICT` (git's word) and the file's name, is alarming, the panel says a push was refused, the second flurry is fenced exactly as today's scenario fences it, then Resume.
  - New: **"An edit you have not committed in a file somebody else changed waits for its commit"** under `commit: manual push: auto`: the person commits one file with the other unticked, the push says which file to commit first, the pill is **not** paused, committing the second file pushes and integrates, `0 unpushed`.
  - "A reload does not clear a stop, and Resume in any tab does" switches to the conflicting step.
- `pinned_git_pause.feature`: switches to the conflicting step; its prose ("nothing here pulls, rebases or forces") is rewritten to "a divergence is taken in; a conflict is what stops it".
- The existing `Then the commit pill explains "rejected"` assertions become `explains "CONFLICT"` where the fixture is a conflict, and are dropped where the push now succeeds.

## 10. Docs, in the same PR

- **[docs/git.md](docs/git.md)**
  - "When it stops": the divergence paragraph is replaced. A divergence is no longer a stop. What stops the loop is a conflict, a fetch or push git refused, or an integration that could not run. The `Overlapped` case is described as a wait, not a stop.
  - "Pushing": "There is no pull, no fetch…" becomes a paragraph on integration: fetch, rebase in a worktree olai owns under `.git`, one two-tree update of your files that applies whole or not at all, never a force, never a stash, your staged and unstaged edits untouched. Then the shape of history (linear; the last-commit sha changes after a rebase) and what a conflict looks like and how to resolve it (`git pull --rebase`, then Resume).
  - "The pill": "the last push was refused" now also covers a refused fetch or a conflict; one sentence.
  - The boot paragraph: a boot takes in the other machine's commits before pushing.
- **[packages/plugins/git/docs.md](packages/plugins/git/docs.md)**: one line under "The door": the row fetches and rebases only on the push path, and owns a temporary worktree under the git directory while it does.
- **[docs/index.md](docs/index.md)**: the one-line description of git.md gains "integration before push" if it lists the features; otherwise untouched.
- **website/**: untouched. The reason to try olai has not changed, and no picture there shows the refusal.

## 11. Residues, named rather than papered over

- A crash (SIGKILL, power) between `read-tree` and `update-ref` leaves the index and files at the rebased tree with `HEAD` one step behind. The next survey shows the upstream's changes as staged edits; under auto the window commits them; the next push's rebase drops that commit as empty (git's default `--empty=drop`). The reverse order was rejected because its residue is a commit that reverts the upstream's changes.
- A commit typed in a terminal in the milliseconds between the rebase reading `HEAD` and `update-ref` is refused by the compare-and-swap and reported, but the tree was updated and put back in that window, which a watcher on the vault may see as two revisions.
- The temporary worktree is a full checkout of the vault under `.git`. Cost is the vault's size on disk, once per push that has something to take in. A sparse checkout would bound it and is not in this PR.
- `git` on the server's PATH is not pinned. Every command used is older than `worktree add --no-checkout` (2.9); the three `-c` keys are ignored by a git that predates them.

## 12. Not in this PR

- Sync (fetching on a schedule, pulling when there is nothing to push).
- A `behind` count on the pill or in the panel.
- Sparse checkout for the temporary worktree; `git replay`.
- Any change to what a commit is, how the window arms, or what Resume does.

## 13. PR mechanics

1. Work on branch `git-pull-before-push` in this worktree. Commit often, in the order of §8 then §9 then §10, each commit building and its own tests green.
2. Plumbing first (`git.ts` + `git.test.ts`), then policy (`pending.ts` + `pending.test.ts`), then the format field and the tool text, then headless, then e2e, then docs. Run `just typecheck-fast-remote` and `just test-fast-remote` per commit; `just e2e-fast-remote` once the features change; `just ci` on the pushed branch at the end.
3. Open the PR with `gh pr create --draft`, titled "Git: take in the upstream before pushing, so a push is never a non-fast-forward". The body: the rule from §1, the rebase-not-merge reasoning from §3 in three sentences, the outcome table from §5, and the residues from §11. No screenshot and nothing from one.
4. Do not merge. Do not mark ready for review. Leave both to the human.
5. This plan file is for approval and is not committed with the PR unless the human says so.

## 14. Decisions the human has taken (2026-09-15)

Asked and answered before implementation begins. These are settled, not open.

- **Rebase rather than merge** (§3). Not `pull.rebase`.
- **No new setting**; integration is part of every push door, boot included (§3, §5).
- **`Overlapped` waits rather than pauses**; a conflict pauses (§5).
- **The rebase runs in a linked worktree under `.git`**, never in the served tree (§4).

What remains for the human is the PR review itself.
