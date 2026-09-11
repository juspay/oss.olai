# fable-delivery-review — the two-PR plan in consolidation.md §7

Token `CORDIS-DELIVERY`. Bounded review of §7 as written 2026-09-11, against the actual pin path between the two repositories (olai `5e087668e`: `nix/kolu.nix`, `nix/cordis.nix`, `scripts/check-hydrated-deps.sh`, `justfile`; kolu `56152455b`: `nix/consumer.nix`, `nix/consumer-closure.json`, `npins/`). Design only; no PR is opened by this review.

## 1. Verdict

The shape is right: two long-lived drafts, phases as commits and review checkpoints, no intermediate merges, kolu first, repin-and-revalidate before the olai merge, and no change to existing `@kolu/surface*` packages so the drishti/odu gate is not triggered. The corrections below are about HOW the olai draft pins the kolu draft, because that path already exists and §7 describes it in words that do not quite name it.

## 2. The pin path as it exists (EXISTING)

- Olai does not vendor `@kolu/*` itself. It imports **kolu's own** `nix/consumer.nix` at the npins revision and asks for named `seeds` (`nix/kolu.nix:62-84`: `@kolu/detect`, `@kolu/solid-dockrow`, `@kolu/surface-app`, `@kolu/surface-cli`, `@kolu/surface-mcp`); kolu walks its `nix/consumer-closure.json` and hands back every member the seeds reach, each with its `external` dependency versions.
- A member kolu cannot ship (a gitignored graft such as `osfacts-client`) is declared in kolu's closure with its pin **revision**; the consumer supplies `pinnedSources.<name> = { src; revision }` and **kolu refuses at eval if the two revisions disagree** (`consumer.nix:32-36, 104-132`; `nix/kolu.nix:88-96`).
- Olai's `just check` runs four hydration legs — `kolu-deps`, `odu-deps`, `odu-surface`, `cordis-deps` (`justfile:46`) — each `scripts/check-hydrated-deps.sh <pin>` comparing olai's root `package.json` against the pin's own answer, overrides included.
- Kolu has npins too (`npins/sources.json`), so a git-revision cordis pin with `@ts-nocheck` hydration is expressible in kolu with the same tool olai uses.

## 3. Corrections to §7

### 3.1 Phase 1's exit evidence should name the closure, not "hydration"

"Reproducible dependency hydration" is true and unfalsifiable. The concrete gate is:

```text
kolu draft, phase 1:
  packages/effect-cordis/          ← moved verbatim, tests included
  npins: cordis @ 00278924a         ← the exact olai pin; @ts-nocheck hydration as nix/cordis.nix does it today
  nix/consumer-closure.json         ← member "@kolu/effect-cordis" { external: { cordis: <graft>, cosmokit, @standard-schema/spec, js-yaml } }
                                      cordis declared as a GRAFTED pin with its revision, the osfacts-client shape

olai draft, phase 1:
  nix/kolu.nix seeds += "@kolu/effect-cordis"
  nix/kolu.nix pinnedSources.cordis = { src = npins.cordis; revision = npins.cordis.revision }   ← kolu refuses a mismatch at eval
  nix/cordis.nix                    ← deleted; olai no longer names the engine anywhere
  justfile: cordis-deps leg retired; the kolu leg now answers for cordis' externals through the closure
  fence: "only effect-cordis names cordis" (fence.test.ts:1155) becomes "no olai package names cordis"; scripts/prove-fence.sh mutation 16 stays red
```

That is the whole of "keep the exact Cordis source pin through existing conventions", and it is checkable: `just check` names any drift, `nix build` throws on a revision mismatch.

### 3.2 A pin to a draft needs the branch on `juspay/kolu`, and the `branch` field changes

npins records `repository.owner/repo`, `branch` and `revision` (`npins/sources.json`). Olai can pin any pushed commit of `juspay/kolu` by revision — including a draft-PR branch — but not a commit on a fork, and the `branch` field will read the PR branch until the final repin sets it back to `master`. §7 should say: the kolu draft branch lives on `juspay/kolu`; each checkpoint repin is `npins update kolu --branch <pr-branch>` at the exact SHA; the post-merge repin returns `branch: master`.

### 3.3 Repin at checkpoints, not per commit; keep a tested-pairs ledger

Every kolu push (and every rebase onto moving kolu master) invalidates the olai evidence. To bound the churn: repin only at phase checkpoints, and carry the record in the PR bodies rather than in memory.

```text
Kolu PR body:   phase 1 = a1b2c3d..e4f5a6b · phase 2 = … · phase 3 = … · phase 4 = …
Olai PR body:   tested pairs — (kolu e4f5a6b, olai 9c8d7e6): just check + just ci green · (kolu …, olai …): …
                current pin: kolu <sha> (branch <pr-branch>)
```

The final row is the "exact commit pair" §7's phase 5 asks for.

### 3.4 Squash is kolu's convention, so plan the post-merge repin as a phase

Kolu's history is squash-merged (`… (#2234)`), so the human's kolu merge WILL produce a new SHA that no olai evidence has seen; a merge commit would have kept the pinned SHA reachable and skipped the step, but that is the human's call, not the plan's. §7's merge gate step 3 is therefore not an "if" but the normal path: repin olai to the master SHA, `branch: master`, rerun `just check` and the full olai CI, record the pair, then the olai merge. Note that the master SHA also carries whatever else merged into kolu meanwhile; the olai run must be green on that revision, which §7 already says.

### 3.5 Kolu-side gates each phase must pass on both CI platforms

Kolu's own CI rule requires two-platform coverage (the `/ci` skill). Phase exits should say "kolu unit + typecheck + nix lanes green on both platforms" — in particular `upstream.test.ts` (the pin-assumption tests) and the new packages' `pnpm-typecheck.nix`/`workspace-tree.nix` builds, since cordis arrives as a hydrated non-npm source and kolu's `fetchPnpmDeps` hash does not cover it. Kolu's `just install`-equivalent needs the same hydrate step olai's has; that is a phase-1 deliverable, not an assumption.

### 3.6 Additive only in kolu's shared Nix seam

`nix/consumer.nix` and `nix/consumer-closure.json` are outside the `.claude/rules/surface.md` glob but are consumed by every downstream that vendors kolu. Adding members and one grafted pin is additive and safe; changing `consumer.nix`'s interface (`seeds`, `pinnedSources` shape) is a downstream-affecting change the gate does not catch. §7 should list "consumer seam: additive members only" beside "existing Surface public APIs preserved".

### 3.7 Ownership

§7 names none. Proposed:

| what | owner |
| --- | --- |
| kolu draft: code, per-phase commits, kolu CI | one implementing agent, on `juspay/kolu` |
| olai draft: migration commits, repins, `just check`/`just ci` | one implementing agent, on `juspay/olai` |
| tested-pairs ledger, checkpoint calls, phase gate reviews (the usual gauntlet, per phase, no merges) | coordinator |
| both merges, and the squash/merge choice | the human |

### 3.8 Phase 4 should consume public doors only

"Build the job board from public package exports" holds only if the example imports the three packages by name through their `exports` maps (kolu's `packages/surface/example/` convention), never by relative path, and the per-door import-closure fence runs on it too. One sentence in phase 4.

### 3.9 One wording fix

Phase 1, olai column: "remove the local implementation and update ownership fences" — the plugin-api door keeps re-exporting the runtime list (`plugin-api/src/runtime.ts:95-123`) but from `@kolu/effect-cordis`; that is the one arrow, not a shim. Say "repoint the runtime door" so nobody reads it as a compatibility re-export to be removed later.

## 4. Disposition

§7 is accepted as the delivery plan with 3.1-3.9 folded; 3.1 and 3.4 are the two that change what an implementer does on day one. No PR is to be opened from this review.

Folded by the coordinator: consumer-closure, hydration and check gates in phase 1; runtime-door re-exports allowed; same-repo draft branch with immutable checkpoint pins; tested pairs in the PR bodies; the post-merge master repin and full olai checks as the normal path; two-platform coverage; additive consumer interface; repo owners and coordinator roles; public-export demo. Two scope choices I accept: the graft package layout is kept as an implementation proof rather than prescribing one raw cordis source, so the consumer delivery must carry every required engine package (`cordis`, `@cordisjs/plugin-loader`, `plugin-include`, `plugin-group`) and its transformations; and no unverified npins command syntax and no blanket per-phase gauntlet mandate were added. Both drafts remain human-merge only. Review closed.
