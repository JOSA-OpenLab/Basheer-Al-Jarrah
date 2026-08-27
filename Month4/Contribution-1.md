# Contribution 1 — go-git

**Target repository:** [go-git/go-git](https://github.com/go-git/go-git)
**Issue:** [#2249 — Stack overflow while processing delta pack](https://github.com/go-git/go-git/issues/2249)
**Pull request:** [#2337 — `plumbing: format/packfile, Break cyclic delta chains instead of recursing`](https://github.com/go-git/go-git/pull/2337)
**Date opened:** 23 Aug 2026
**Status at time of writing:** open, CI queued pending maintainer approval (first-time contributor)
>[!note] note:  The maintainer merged the PR one day after it was opened

> Everything under "Observed" was read directly off GitHub or the repository on **23 Aug 2026**. Anything marked *My read* is my own judgment, not a claim by the project.

---

## 1. The project

**go-git** is a Git implementation written in pure Go. Rather than shelling out to the `git` binary, it reads and writes real `.git` directories and speaks the Git wire protocols natively, exposing both *plumbing* (low-level) and *porcelain* (high-level) APIs.

**Why that matters:** shelling out to `git` means depending on a binary existing at a compatible version, parsing human-readable CLI output that is not a stable API, and receiving errors as exit codes and stderr text. For a Git *hosting* platform or a continuous reconciliation loop, that is an operational liability. go-git replaces it with Go types and `error` values.

**Observed — repository health (23 Aug 2026):**

| Metric | Value |
| --- | --- |
| Stars | 7,677 |
| Last commit | 2026-08-22 |
| Open PRs | 96 |
| `good first issue` / `help wanted` | 18 / 53 |
| External non-bot PRs merged since 1 Jun | 40 (median 8 days to merge) |
| Maintainer review rate on recent external PRs | 12 / 12 |

**My read:** the defining design decision is that storage is abstracted behind interfaces (`Storer`) and the filesystem behind `go-billy`, so a repository can live entirely in RAM. That is why it works in serverless functions and WASM, and it is also why the test suite is fast and hermetic — which is what made this contribution feasible without any infrastructure.

Downstream users include Gitea, Flux (CNCF), Kubernetes Prow, Pulumi, and Keybase. This is load-bearing infrastructure, which explains the project's strict contribution posture.

### Architecture, and where the issue sits

```
Repository.Push
  └─ plumbing/transport (send-pack)
       └─ plumbing/format/packfile.Encoder          builds the packfile to upload
            └─ DeltaSelector.ObjectsToPack          decides what is stored as a delta
                 ├─ objectsToPack()                 loads objects from the storer
                 │    └─ fixAndBreakChains()        <-- the bug
                 └─ walk() / tryToDeltify()         finds NEW deltas in a sliding window
```

A packfile stores most objects as **deltas** — instructions to rebuild object X from a base object Y. Two hard constraints follow: a base must be written *before* any delta depending on it, and every chain must terminate at a whole object.

`DeltaSelector` has two jobs. `walk`/`tryToDeltify` *create* new deltas. Separately, when objects come from an existing local packfile, go-git **reuses** the deltas already stored there instead of recomputing them — a significant performance win. `fixAndBreakChains` is the reconciliation step for that reuse path: each reused delta knows its base only as a *hash*, so this function resolves those hashes into object pointers and, where a base is unavailable, "breaks the chain" by undeltifying. Hence the name.

---

## 2. Finding the issue

This took longer than the fix. go-git's attractive `good first issue`s are heavily contested, so most candidates had to be discarded.

**Method.** I queried the GitHub API for open, unassigned issues labelled `bug` / `good first issue` / `help wanted`, then cross-referenced each against every PR touching it. My first pass only checked *open* PRs, which was a mistake — a merged PR against a still-open issue is invisible that way. I rebuilt the screen to check PRs in **any** state plus referenced commits.

**Eliminated as already claimed:**

| Issue | Blocked by |
| --- | --- |
| #1917 (race in `Tree.FindEntry`) | PRs #1978, #2029 |
| #1822 (`RemoveReference` empty dirs) | PRs #2052, #2030 |
| #1190 (`RemoveGlob`) | PR #1950 |
| #2196 (`feature.manyFiles`) | PR #2199 |
| #1787 (Prune panic) | PR #2283 |
| #1911 (`PatchContext` timeout) | PR #1910 (linked from issue body, not the timeline) |
| #417 (`LogOrderCommitterTime`) | PR #2288 |

**Eliminated on verification** — these were the instructive ones:

- **#530** (`refs/pull` merge target) — *already fixed* on `main`; the validation had been removed with a comment explaining git's real behaviour. The issue was simply stale.
- **#1526** (`PackSession.Handshake` leak) — largely fixed by merged PR #1635; the file had been refactored.
- **#554** (`missing 'want ' prefix`) — I reproduced it, then rejected it. The v6 server already peeks for the flush-pkt before decoding, and the pack-protocol grammar requires at least one `want` line, so a "fix" would have been arguing against the spec.
- **#2155** (`core.repositoryformatversion`) — I had this as my top recommendation and was wrong. PR #2167 was a backport to `releases/v5.x` of a fix already present in v6; the issue only *looked* open because GitHub does not auto-close from a non-default branch.

**Lesson recorded:** an open issue is not evidence of an unfixed bug. Every candidate has to be verified against the current default branch before any work starts. Four of my candidates died this way.

---

## 3. The bug (#2249)

`fixAndBreakChainsOne` resolves a reused delta by recursing into its base first, and assigning `ObjectToPack.Base` only *after* the recursive call returns:

```go
if otp.Base != nil {        // the only reentry guard
    return nil
}
...
if err := dw.fixAndBreakChainsOne(objectsToPack, base); err != nil {  // descend
    return err
}
otp.SetDelta(base, otp.Object)                                        // Base assigned here
```

**Root cause: a temporal gap between the descent and the assignment.** `Base` is the sole marker that an object has been visited, but it is set *after* recursing. So while the recursion is in flight, every object on the current path still has `Base == nil` and is indistinguishable from an unvisited one.

Given a cycle `A -> B -> A`, the guard never fires:

| depth | object | `otp.Base` | action |
| --- | --- | --- | --- |
| 1 | A | nil | base B found → descend |
| 2 | B | nil | base A found → descend |
| 3 | A | **still nil** | base B found → descend |
| … | | | until the stack is exhausted |

The result is `fatal error: stack overflow` — unrecoverable, since `recover` cannot catch stack exhaustion. It aborts the whole process during a push.

**On the reporter's comment.** They wrote that there is "no recursive protection in `fixAndBreakChainsOne`, so the 2139126 commit may not be final fix." They were right, and checking this mattered. Commit `2139126` is `storage: filesystem, MRU pack ordering` — it changes which packfile is consulted first, so at most it perturbs whether a given repository happens to produce a cyclic delta set. It touches nothing in the recursion. So the trigger was masked; the defect was not. This is why I fixed the recursion itself rather than a symptom.

**Reproduction.** I built a two-object cycle and confirmed the crash on `main` @ `374c3548`. The stack trace matched the reporter's frame-for-frame — entry at `:178`, then `:185` repeating between two alternating object pointers — which is what convinced me I had the actual reported defect and not a lookalike.

---

## 4. The fix

Track the objects on the current resolution path; mark the current object *before* inspecting its base; if the base is already on the path, break the chain by undeltifying instead of following the cycle.

```go
// visiting holds the objects on the current resolution path, so that a
// delta chain looping back on itself can be detected and broken.
visiting := make(map[plumbing.Hash]bool)
...
h := otp.Hash()
visiting[h] = true
defer delete(visiting, h)

if visiting[do.BaseHash()] {
    return dw.undeltify(otp)
}
```

**+204 / −3 across 2 files.**

### Design decisions

**The check goes before the descent, not on entry.** My first draft got this wrong. Checking "am *I* already being visited?" on entry looks equivalent but is not: the inner frame undeltifies the object, then the *outer* frame — still holding its own `base` local — falls through to `SetDelta` and re-creates the loop. Testing whether the **base** is on the path means the cut is made by the frame that owns the offending edge, and no stale frame can re-link it.

**The path set is unwound on return.** `defer delete` is required for *correctness*, not tidiness. Without it, two deltas legitimately sharing one base would be seen as a cycle and the second needlessly undeltified — a silent pack-size regression. Two tests exist purely to pin this down.

**Break the chain rather than return an error.** This matches what the function already does when a base is missing or is not a delta object — it is literally named `fixAndBreak*Chains*` — and it keeps a push working instead of failing it.

**Rejected — a depth limit.** It would stop the crash but cannot distinguish a legitimate 60-long chain from a 2-cycle, so it would either truncate valid chains or still allow thousands of pointless frames. Cycle detection is exact at the same cost.

**Rejected — converting recursion to iteration.** It would also remove deep-stack risk, but it is a larger rewrite of working logic for a bug about *cycles*, not depth. It would enlarge the diff against the project's "no unnecessary abstractions" guidance. Worth mentioning to a maintainer as a follow-up, not smuggling into a bug fix.

### Alignment with upstream git

After implementing, I checked git's own [`break_delta_chains()`](https://github.com/git/git/blob/master/builtin/pack-objects.c#L2490). It uses a three-state DFS and does the same three things in the same order:

```c
cur->dfs_state = DFS_ACTIVE;                     /* mark self first */
if (DELTA(cur)->dfs_state == DFS_ACTIVE) {       /* is the BASE active? */
        drop_reused_delta(cur);                  /* undeltify this object */
        cur->dfs_state = DFS_DONE;
        break;
}
```

git's comment confirms the ordering is deliberate — *"It's important to do this `_before_` we loop, because it impacts where we make the cut"* — and explains why cutting the **last** edge is preferable: for `A -> B -> C -> D -> B`, cutting `D->B` keeps A's depth correct at 3, whereas cutting `B->C` drops C and D from the chain.

**My read:** this was the single most valuable step. go-git's stated goal is matching reference git, so "this is the approach git uses" is a far stronger argument than "this seems correct to me." `drop_reused_delta` is also the direct counterpart of go-git's `undeltify` on the reuse path — which independently supports the claim that the delta-*reuse* path is where cycles are expected to originate.

---

## 5. Tests

New file `plumbing/format/packfile/delta_selector_cycle_test.go` (179 lines), following the existing `testify/suite` convention in the package. Objects are written into a real `memory.Storage` so `undeltify` can genuinely restore originals.

| Test | Covers |
| --- | --- |
| `TestSelfReferencingDelta` | `A -> A`; only escape is undeltifying |
| `TestTwoObjectCycle` | the reported case, `A -> B -> A`; asserts exactly one object undeltified |
| `TestThreeObjectCycle` | `A -> B -> C -> A`; confirms the cut lands on the last edge (2 deltas kept) |
| `TestAcyclicChainIsPreserved` | shared base — guards against false positives |
| `TestDeepChainIsPreserved` | 50-long valid chain; every delta must survive |

A shared helper walks the resulting `Base` pointers and fails on a repeated hash or a delta left without a base, so the tests assert the output is a *writable* chain rather than merely that nothing crashed.

**The check that mattered most: I verified the tests fail without the source change.** Reverting `delta_selector.go` and re-running kills the test binary with `fatal error: stack overflow`. A regression test that passes either way is worthless, and go-git's AI policy explicitly rejects "AI-generated tests that do not actually exercise the relevant behaviour."

**Verification run before pushing:**

- `make validate-lint` — golangci-lint **v2.13.1**, the exact version pinned in the Makefile: **0 issues**. It initially flagged a `staticcheck` QF1008 warning in my test file; CONTRIBUTING requires zero warnings, so this would have failed CI.
- `go test -race ./...` — all **69 packages** pass, exit 0, no data races.
- `gofmt -l` clean, `go vet` clean.
- Reference `git` 2.55.0 consulted for expected behaviour.

---

## 6. Following the contribution guidelines

go-git is stricter than most projects, and much of the work was in compliance rather than code.

**`AI_POLICY.md`** — the project explicitly welcomes AI-assisted contributions but binds them: disclose via an `Assisted-by:` trailer, own every line ("the AI generated it" is not an acceptable review answer), regression tests required, verify against reference git with upstream links, and **respond to review personally** — no piping feedback back through a model. It also requires designing non-trivial changes in an issue first, which is part of why I chose an already-scoped bug over a larger architectural one.

**Commit message** — `AGENTS.md` specifies `<package>: <subpackage>, <what changed>. [Fixes #N]` with **no blank line after the title** and a blank line before trailers. That quirk is easy to get wrong.

**Pre-validated every CI gate locally against its actual regex** before pushing:

| Gate | Result |
| --- | --- |
| `check-commit-message` package prefix | pass |
| `check-dco` — `^Signed-off-by: .+ <.+>$` | pass |
| `validate-dirty` | pass |
| `validate-lint` | 0 issues |

Also included, per CONTRIBUTING: a **minimum working Go example** reproducing the bug (a hard requirement most PRs miss), DCO sign-off, and targeting `main` rather than `releases/v5.x` per the branch policy.

**Deliberately not done:** no "please review" comment on the issue. `Fixes #2249` auto-links it, and the AI policy warns that low-effort noise exhausts maintainer attention.

---

## 7. Open risks

Flagged openly in the PR rather than glossed over, since a maintainer will find them anyway:

1. **Reachability.** The test constructs the cycle through the internal API, not from a crafted packfile. I did not claim a proven remote trigger; I framed it as defence-in-depth on the delta-reuse path, with the reporter's production stack trace as the field evidence. That upstream git guards this exact scenario (`drop_reused_delta`) is the supporting argument.
2. **Break vs. error.** A maintainer may prefer surfacing a cycle as an error, on the view that it signals a corrupt local pack worth reporting. Small change if so.
3. **Signature change.** `fixAndBreakChainsOne` gains a parameter. Unexported with one caller, so no API impact, but they may prefer the state threaded differently.
4. **`defer` in a recursive function.** Negligible here, but a performance-minded reviewer might flag it on a large-push path.
5. **Backport.** `releases/v5.x` appears to have the same shape. I offered a backport rather than presuming, since the project states v5 capacity is scarce.

---

## 8. What I learned

- **Verify against the default branch before writing anything.** Four candidate issues were already fixed, partially fixed, or spec-incorrect. Screening on labels and comments alone would have wasted the whole effort.
- **Cross-reference PRs in every state, not just open ones.** My first screen missed a merged backport, and I only caught it when it was pointed out. Also worth noting: a PR linked from an issue *body* produces no timeline event, so it is invisible to a timeline-only check.
- **Reproduce before diagnosing, then match the reporter's evidence.** Getting a frame-for-frame match with the reported stack trace turned "a plausible bug" into "the reported bug."
- **A negative control is not optional.** Proving the test fails without the fix is the difference between a regression test and decoration.
- **Read the project's own conventions and CI as source material.** Running the *pinned* linter version caught a warning that would have failed the build; reading the workflow YAML let me check every gate locally against its real regex.
- **Cite the reference implementation.** In a project whose goal is git compatibility, showing that git solves it the same way is the strongest argument available — and it caught nothing wrong, but it did confirm my cut-point choice was the optimal one for chain depth rather than merely a working one.

---

## 9. Next steps

- Wait for a maintainer to approve the queued workflow runs (required for a first-time contributor) and confirm CI is green across Linux/macOS/Windows and Go oldstable/stable.
- Respond to review feedback **in my own words**, as the AI policy requires.
- If accepted, offer the `releases/v5.x` backport.
