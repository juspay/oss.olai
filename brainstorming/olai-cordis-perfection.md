# Making Olai's plugins safe to stop and replace

*Reviewed against Olai `cdbc1462b`, 2026-09-06. Findings by Claude Fable 5.1 and GPT 6 Astra; rewritten by Astra at the user's request. This is a proposed work list, not authorization to implement it.*

The question behind this work is simple: **can you turn off one plugin, or replace something it uses, without leaving its work running or breaking unrelated plugins?**

## Two implementation phases, one PR each

**Proposed by Fable and GPT 6 Astra after debate, under the user's review model:** both agents review each PR; after refactoring and green CI, the user takes the final look and merges. Minimize PR count; individual findings need their own evidence, not their own merge.

| Phase | PR | Sections | Result |
| --- | --- | --- | --- |
| 1 | Make plugin shutdown and cancellation safe | [1](#1-do-not-call-a-plugin-after-it-has-finished-stopping), [7](#7-closing-a-terminal-pane-must-stop-everything-it-created), [8](#8-cancellation-must-not-leave-unintended-files-staged-for-git), [9](#9-give-background-chat-work-a-lifetime-and-a-cancellation-path), [11](#11-acquiring-a-resource-must-also-guarantee-its-cleanup), [13](#13-keep-duplicate-service-errors-useful-when-cordis-changes), [14](#14-document-the-bridges-upstream-assumptions-accurately); [6](#6-establish-what-browser-clients-promise-across-reconnection) verification and contract documentation | Stop late callbacks and background work, close acquired resources, protect Git recovery, and guard the bridge's upstream assumptions. Establish reconnection behavior before changing browser ownership. |
| 2 | Make shared state and dependencies explicitly owned | [2](#2-give-shared-browser-state-an-actual-service-owner), [3](#3-remove-access-to-the-browsers-private-composition-machinery), [4](#4-stop-retaining-browser-services-after-their-owners-leave), [5](#5-declare-optional-service-access-instead-of-allowing-arbitrary-lookups), [10](#10-give-the-floating-menu-container-an-owner), [12](#12-make-the-dependency-checks-catch-shared-live-state); [6](#6-establish-what-browser-clients-promise-across-reconnection) implementation only if needed | Put live values on services, remove private-runtime and unrestricted lookup paths, give the overlay an owner, and enforce the boundary with dependency checks. Preserve working reconnection behavior. |

**Why two instead of one?** A broad ownership migration may need to be reverted. Its revert should not also restore the terminal leak, late callbacks or Git cancellation problems fixed in phase 1. Two PRs also allow those repairs to ship before the migration is complete. No technical dependency requires a separate merge; this split earns its cost through rollback separation and earlier delivery. With squash merging, each PR is one revert unit; its internal commits do not remain separate revert units on the main branch.

**Why not more?** Both reviewers can inspect focused commits and regression evidence within a PR. Additional merge boundaries add coordination and validation cycles without another identified benefit. Keep docs, dependency checks and compatibility tests with the changes they govern. Phase 1 is not presumed easy: event shutdown and Git outcome recovery need careful proofs.

**Order inside the work:** establish regressions and reconnection behavior first; repair invocation and task lifetimes and acquisitions; then migrate service access and add its checks. Do not repair holders merely to delete them in phase 2 unless a concrete delay makes an interim fix worthwhile. Phase 2 may be prepared alongside phase 1, but its final review must include the merged phase-1 result. There is no mandatory observation period between merges.

**Review and merge:** GPT 6 Astra and Fable independently review the assigned scope and regression evidence, leaving attributed PR comments. The implementor addresses findings and requests another review until both are satisfied. After the refactoring pass below, both reviewers check the resulting changes and refresh their verdicts on the final commit; affected checks and required CI must pass on that head. The user then takes the final look and merges. Agents do not enable automatic merging or merge the PR themselves.

Cordis helps answer that question through three mechanisms:

- A **service** is something one plugin makes available to others, such as file access or navigation. A **key** names that service.
- A plugin's **declared dependencies** tell Cordis which services it needs. Cordis starts the plugin when they are available and stops its dependent work when they leave.
- A **scope** records cleanup actions: remove a listener, close a connection, stop background work. Each activation—one run between starting and stopping—has its own lifetime.

These mechanisms need to cover the actual work, not just the registration of a plugin. They also have limits: stopping a plugin should close its resources, but should not undo a person's saved edits or retract messages already delivered. The [paper's system boundary (§6.1)](https://arxiv.org/pdf/2608.25512) makes that distinction.

Below, **reproduced** means a probe demonstrated the failure; **source finding** means the code exposes the problem but the workflow still needs a regression test; **design concern** means the ownership needs improving without claiming a demonstrated user-facing failure; **verification task** means establish the behavior before deciding whether to change it. Code fragments illustrate the concern or desired shape; they are not complete patches.

## 1. Do not call a plugin after it has finished stopping

**Proposed and reproduced by GPT 6 Astra; probe re-run and sources confirmed identical by Fable. Status: reproduced.**

An event can be delivered to several plugins. Olai saves the recipient list before delivering it. If the first recipient takes time, a later recipient can finish stopping before its turn arrives—and still get called.

The probe produced this sequence:

```text
second resource released
second.dispose completed
second handler called; resource alive = false
```

The cause is the saved list in `packages/effect-cordis/src/broadcast.ts`:

```ts
Effect.forEach(handlers.read(), handler => handler(value))
```

Handler errors are caught and logged by `contained`; one thrown error does not propagate through the broadcast. That does not make a late call harmless: it can perform an unwanted action before throwing, or without throwing at all. The reproduction proves execution after disposal, not a particular user-data loss.

**Change:** stopping a listener must prevent calls that have not started and safely cancel or finish calls already running before their resources close. Do not merely wait forever for them: a hung handler, or one that requests its own removal, needs an explicit policy.

**Proof:** keep the two-plugin probe as a regression, then cover removal during an already-running handler. Check `waterfall.ts` too: it saves a similar list, although only the broadcast failure has been reproduced. The review probe is temporarily available at `/tmp/olai-cordis-current-audit/race.ts`; put a permanent version in the repository.

## 2. Give shared browser state an actual service owner

**Proposed by Fable and GPT 6 Astra. Status: design concern, confirmed in source.**

Some plugins announce “my browser state is ready,” but pass the state itself through imported global variables. Cordis sees the announcement, not all the consumers of the state.

Outlines illustrates the split in `packages/plugins/outlines/src/browser.tsx`:

```ts
holdUndo(undo)                  // real value goes into a module variable
Offers.own("browser-state", () => ({})) // service carries only readiness
```

Outline references read by chat and Markdown actions read by journal use similar shared signals. Their current cleanup includes useful guards; this is not a claim that all those guards fail.

**Change:** put the values on the service that already names them. Consumers should receive that service through their declared dependencies:

```ts
const state = yield* browserState
state.undo.clear()
```

Only the integration using a departed service needs to stop. Chat itself should remain available when outline references disappear. Internal helpers can remain, provided their lifetime is tied to the correct activation.

**Proof:** withdraw and restore each provider while its consumers remain mounted. Verify fresh state and preserved unrelated work.

## 3. Remove access to the browser's private composition machinery

**Proposed by Fable. Status: design concern, confirmed in source.**

Some plugins obtain rendered contributions by importing the module that assembles the whole browser application. That bypasses the public service intended for this purpose.

For example, `packages/plugins/layout/src/Mounted.tsx` imports:

```ts
import { hung } from "@olai/web/client/plugins/runtime.ts"
```

**Change:** obtain the contributions through `Faces` or the appropriate renderer service, declared in `needs`, and pass that value to the component. Restrict package exports so plugins cannot reach private runtime files. Preserve exports for genuinely shared controls and utilities.

**Proof:** a dependency check rejects importing the private runtime; renderer withdrawal still removes the dependent UI correctly.

## 4. Stop retaining browser services after their owners leave

**Proposed by Fable. Status: source finding.**

Several helpers remember a live service but never forget it when the plugin stops. For example, chat's browser wire uses:

```ts
export const holdChatWire = (read: () => ChatClient): void => {
  held = read
}
```

Chat's `holdFaces`, and wire holders in git and journal, need the same audit. Keeping the reference is demonstrated in source; a stale reference is not by itself proof that a particular user action can still invoke it.

**Change:** while these holders remain, return and register cleanup that clears only the value installed by that activation:

```ts
held = read
return () => { if (held === read) held = undefined }
```

Prefer receiving the service directly as part of the shared-state work in section 2.

**Proof:** after stopping the owner, its helper cannot return the old service. An old cleanup must not clear a replacement activation's value.

## 5. Declare optional service access instead of allowing arbitrary lookups

**Proposed by Fable; qualified by GPT 6 Astra. Status: design concern.**

The MCP plugin declares `HostServices`, then uses it to look up services it did not individually declare:

```ts
needs: [TransportSurface, HostServices, Offers]
// later:
services.current(Directory)
services.current(Ops)
```

That makes the real dependency graph harder to see and govern. Browser `readService` and vault setup's ledger/search lookups need the same review.

**Change:** replace unrestricted lookup with explicit dependencies or a declared service that manages optional availability. One possible design gives MCP a transport component and a separate component for vault-dependent tools. Another is a narrow broker: a service whose documented job includes handling the arrival and departure of its backing providers.

Optional behavior must survive the change: **MCP must work without a vault, and the vault must work without git.** Do not introduce a dependency cycle by mechanically making every lookup mandatory.

**Proof:** test those independent configurations, plus adding and removing the optional providers while the process remains running.

## 6. Establish what browser clients promise across reconnection

**Proposed by Fable; changed to a verification task after GPT 6 Astra's review. Status: verification task, not a proven defect.**

`Wired` gives plugins access to their server clients. On a roster change, the connection can redial while surviving browser plugins stay mounted:

```ts
await live.redial(surfaceMapOf(halves))
await composeTo(halves, plugin => live.clients[plugin])
```

This is legitimate if `Wired` promises a stable client whose subscriptions and calls follow the connection. A service can manage changing connections internally; the paper permits that broker design (§6.2).

**First prove:** keep a subscription open, switch an unrelated plugin, and verify that updates and new calls still work. A removed capability must refuse further calls. Preserve unrelated editor elements and drafts.

**Then decide:** if that behavior works, document and test the contract. If it fails, investigate repairing the broker before choosing to replace the service and restart its dependents. A failing test alone does not establish which design is right. Restarting every plugin on every redial is not a requirement of this audit.

## 7. Closing a terminal pane must stop everything it created

**Proposed by Fable; cleanup sketch corrected by GPT 6 Astra. Status: source finding.**

The Kolu terminal pane loads its terminal library asynchronously, then registers cleanup:

```ts
onMount(async () => {
  const library = await import("@xterm/xterm")
  // create terminal and resize observer
  onCleanup(/* dispose them */)
})
```

In `packages/plugins/kolu/src/appliance/props/LivePane.tsx`, the cleanup registration happens after Solid has left the component's ownership context. Closing the pane therefore does not reliably dispose what was created.

**Change:** register cleanup synchronously, mark the component disposed when it runs, and check that flag after loading before creating anything. Also handle import failure and clear the delayed callback whose timeout handle is currently discarded.

```ts
let disposed = false
onCleanup(() => { disposed = true; releaseCreatedResources() })
// inside the async continuation:
await loadLibrary()
if (disposed) return
```

**Proof:** close the pane both before and after the library loads. Neither case may leave a terminal, observer or delayed callback running.

## 8. Cancellation must not leave unintended files staged for Git

**Proposed by Fable; outcome handling corrected by GPT 6 Astra. Status: source finding.**

Olai temporarily stages paths to make a commit. It backs up the user's Git index and restores it on ordinary command failure. But the restore calls are ordinary statements: cancelling the operation can skip them.

The relevant sequence in `packages/plugins/git/src/git/git.ts` is:

```ts
const index = keptIndex(placed)
const staged = yield* git(root, ["add", ...])
// manual restore on failure; more asynchronous work follows
```

**Change:** make cleanup part of this individual commit operation's scope. Restore the backup if staging failed or the operation stopped before committing; discard it once the commit has succeeded.

Do not use `Exit.isSuccess` as the decision: this function returns `{ _tag: "Failed" }` as an ordinary result, so Effect success does not mean Git success. A boolean updated after the subprocess returns is also not the whole solution: cancellation can arrive after Git changes HEAD but before Olai observes success. Establish the outcome safely before choosing recovery. Consider concurrent external Git use; having a backup does not give Olai exclusive ownership of the index.

**Proof:** interrupt around staging and committing, and exercise ordinary Git failure. Verify HEAD, the index and backup files—not just the returned message.

## 9. Give background chat work a lifetime and a cancellation path

**Proposed by Fable; ownership distinctions corrected by GPT 6 Astra. Status: source findings requiring targeted regressions.**

Chat starts some asynchronous work without keeping a handle or attaching it to a scope. One example in `packages/plugins/chat/src/scoped.ts` is:

```ts
Effect.runFork(Effect.catch(relocateRoot(), relocationFailed))
```

That work can continue while the node, session or plugin it belongs to is stopping. The detached boot task and several tasks in `chat.ts` need the same audit.

**Change:** identify the right owner for each task, then make its shutdown cancel or join the task. Use the bridge's `detached` helper for work whose result nobody needs; retain a fiber handle where callers need to cancel or wait for it. Route the bare warning-log forks through the appropriate runtime too, so they retain logging settings.

The idle timer is **already manually owned**: `slot.timer` holds it and slot cleanup interrupts it. Do not classify that timer as a leak merely because it uses `runFork`.

**Proof:** stop a session or plugin during relocation and boot. No new session or node resources may appear after shutdown completes.

## 10. Give the floating-menu container an owner

**Proposed by Fable. Status: source finding; small DOM residue.**

`packages/web/src/client/overlay.ts` appends a container to the page and remembers it forever:

```ts
root = document.createElement("div")
document.body.append(root)
return root
```

Its current callers belong to outlines. Turning outlines off leaves the container behind. An empty div is less serious than a running callback or child process, but its lifetime is still undefined.

**Change:** let outlines, or a shared renderer service, create and remove the container within its scope. Choose the owner from actual consumers; a layout should not own it merely because it draws a frame.

**Proof:** stop and restore the owner repeatedly. Containers must not accumulate, and unrelated overlays must remain usable.

## 11. Acquiring a resource must also guarantee its cleanup

**Proposed by Fable. Status: MCP source finding; subprocess cancellation needs verification.**

The MCP endpoint first awaits creation of an adapter, then records how to close it:

```ts
const served = yield* Effect.promise(() => serveSurfaceAsMcp(options))
yield* Effect.addFinalizer(() => Effect.promise(() => served.close()))
```

Cancellation can occur while the promise is running, leaving a created adapter without registered cleanup.

**Change:** acquire and register release together:

```ts
const served = yield* Effect.acquireRelease(
  Effect.promise(() => serveSurfaceAsMcp(options)),
  served => Effect.promise(() => served.close()),
)
```

Acquisition is then protected from interruption until cleanup is registered; it must still have a bounded completion or cancellation strategy. Audit the Odu probe's subprocess ownership as a separate case: a plain promise does not automatically cancel its child process when its caller stops.

**Proof:** stop during acquisition and verify the adapter or child process is closed, including late completion and failure paths.

## 12. Make the dependency checks catch shared live state

**Proposed by Fable; scope refined by GPT 6 Astra. Status: verification improvement.**

The existing dependency check—called the “fence”—rejects sockets and timers in public contract modules, but permits a live service hidden in an exported module variable:

```ts
let held: Client | undefined
export const current = () => held
```

**Change:** test for shared activation state crossing package boundaries outside declared services. Inspect syntax and imports, with fixtures for the specific prohibited patterns. Cover mutable objects declared with `const` as well as `let` and Solid signals.

Do not ban every top-level `Map` or `Set`: an inert lookup table can be a valid contract. Nor does a TypeScript `ReadonlyMap` annotation prove runtime immutability. A source check supports the ownership rule; it cannot prove arbitrary program behavior.

**Proof:** known bad examples fail the check and legitimate static contracts pass. Introduce those tests alongside section 2 so the refactor has a guard against regression.

## 13. Keep duplicate-service errors useful when Cordis changes

**Proposed by Fable. Status: compatibility-test improvement.**

Olai identifies Cordis's duplicate-provider error by matching its wording:

```ts
cause.message.startsWith(`service "${key.cordis}" has been registered at <`)
```

If a future Cordis version changes that sentence, Olai still gets an error but may lose its structured report of the conflicting owner.

**Change:** add a focused test that offers the same key twice and asserts an `OfferConflict` with the original owner's identity. Ask upstream for a typed error instead of relying on prose.

**Proof:** the test exercises the actual pinned Cordis runtime, not a fabricated error string.

## 14. Document the bridge's upstream assumptions accurately

**Proposed by Fable; qualifications by GPT 6 Astra. Status: documentation and upstream follow-up.**

The bridge depends on particular Cordis behavior. An upgrade needs a checklist of those assumptions, not a claim that every API reference is a bug.

The most consequential example is cleanup ordering:

```ts
// Pinned Cordis unloads its disposer set concurrently:
Promise.all(/* disposers */)
```

Waiting for dependents inside just one of those disposers does not keep the other disposers from releasing resources first. Olai compensates in `lifecycle.ts` by revoking service offers and joining dependent cleanup before releasing provider resources.

**Change:** prepare an upstream reproduction for that ordering problem. Document private/internal assumptions separately from ordinary public API use: disposer ownership, lifecycle polling, loader access, provider identities, events, symbols, error wording and loader update ordering. Correct the stale resolver path in `nix/cordis.nix` from the bundle package to `packages/effect-cordis/src/loader.ts`.

Explain initialization cancellation as Olai's Effect-backed adaptation. Do not claim it is automatically covered by the paper's inertial asynchronous model. Likewise, the bridge records cleanup actions; it cannot prove that authors supplied correct inverses or that all shared operations commute.

**Proof:** documentation names the actual code and tests that guard each fragile assumption. Retain the existing regression that proves dependent cleanup sees a live provider.

## Implementation

1. **Astra launches the implementor.** Use the [Kolu TUI workflow](https://github.com/juspay/kolu/blob/master/agents/.apm/skills/kolu/TUI.md) to create a fresh worktree and terminal for each phase. Refresh the base checkout first without overwriting local work. Launch:

   `kolu create --toplevel --repo ~/code/olai --worktree cordis-phase-N --intent "Olai Cordis phase N" -- claude --model 'opus[1m]' --dangerously-skip-permissions`

2. **Send only a short pointer.** Observe the terminal is ready, send the following with the phase and Astra's current terminal ID filled in, wait for input to settle, then send Enter separately:

   > Read ~/code/oss.olai/brainstorming/olai-cordis-perfection.md and implement phase N. If you have questions at any stage, or when your PR is ready, contact Astra at TERMINAL_ID via the Kolu CLI: https://github.com/juspay/kolu/blob/master/agents/.apm/skills/kolu/TUI.md. Astra and Fable will collaborate and reply.

3. **Implement and report back.** The implementor follows the assigned sections and repository `AGENTS.md`, adds regression evidence, updates docs, and commits, pushes and opens one PR. It sends Astra the PR URL and head commit through Kolu. Use short file pointers for long messages; contact the coordinator directly instead of requiring polling. If questions arise at any stage, the implementor contacts Astra through the Kolu CLI. Astra and Fable discuss the question, agree on an answer, and Astra sends it back through Kolu. Escalate to the user only when the answer requires a new product or scope decision.

4. **Review until both reviewers are satisfied.** Astra coordinates with Fable (currently `755915a7-80ce-4bbc-9eb1-35e8c4a49bfb`; verify the terminal before dispatch). Both review product code only, reconcile privately, and post one combined, attributed PR comment. They do not review tests or make test-code changes a condition of approval; CI results remain evidence. Only the implementor changes product code. It pushes fixes and contacts Astra again. Repeat until both explicitly accept the same commit.

5. **Refactor, then validate.** The implementor follows [The spacetime of code](https://kolu.dev/blog/hickey-lowy/): run separate Hickey and Löwy review passes for tangled concepts and independently changing concerns, weigh their findings, and refactor within the PR's scope. Repeat those passes until no justified findings remain. Run CI and fix failures. Notify Astra of the resulting commit so both reviewers can recheck the changes and refresh their verdicts; earlier acceptance does not cover a later refactor.

   **Cordis adherence is a constraint on every refactor.** Preserve declared service dependencies, per-activation ownership, scoped cleanup, optional-provider behavior, atomic registration and reconnection guarantees. For each proposed product change, identify the affected owners, dependencies and lifetimes, and explain why those guarantees still hold. Reject simplifications that obscure or weaken them. If no refactor is justified, report that rather than forcing a change. Both reviewers check Cordis adherence again on any changed head.

6. **Hand over to the user.** Once final-head CI is green and both reviewers are satisfied, Astra reports the PR ready for the user's inspection and merge. No agent merges it or enables auto-merge.
