# Contribution 2 — go-git

**Target repository:** [go-git/go-git](https://github.com/go-git/go-git)
**Issue opened:** [#2350 — `plumbing/revlist: ObjectsWithRef walks the object graph once per want, but UploadPack only needs membership`](https://github.com/go-git/go-git/issues/2350)
**Pull request:** [#2351 — `plumbing: transport, Walk the object graph once per upload-pack request`](https://github.com/go-git/go-git/pull/2351)
**Date opened:** 29 Aug 2026
**Status at time of writing:** open, CI 17/17 green, awaiting maintainer review
**Base commit:** `52f84ef3`
**Local working copy:** `~/Desktop/MyStuff/Programming/JOSA/oss-contrib/go-git`

> Everything under "Observed" was read directly off GitHub or the repository on **29 Aug 2026**. Anything marked *My read* is my own judgment, not a claim by the project.

---

## 1. Goal for this contribution

Contribution 1 was a correctness fix (an unbounded recursion). This time the goal was explicitly a **performance problem** — an algorithmic improvement, an N+1, or similar.

Two things had changed since contribution 1, both in my favour:

- **PR #2337 was merged one day after opening.** That promoted me out of GitHub's "first-time contributor" state, so CI now runs automatically instead of waiting for a maintainer to approve the workflow.
- I had a working repository, a validated toolchain, and a screening method that had already been corrected once.

---

## 2. Looking for a performance issue — and not finding one

**Observed:** go-git has 8 open issues labelled `performance`. I screened all of them:

| Issue | Why rejected |
| --- | --- |
| #1821 Multi-pack-index (MIDX) | Maintainer roadmap feature, needs design |
| #1820 Refactor iterators | Roadmap item, depends on #1819 |
| #1673 Huge peak memory usage | Assigned to pjbgf, open PR #2069 |
| #1601 System freezes cloning linux | Assigned to pjbgf |
| #181 Status() slow with untracked files | Taken by open PR #2282 |
| #1379 Status with many ignored files | Taken by open PR #2054 |
| #14 Blame is very slow | Addressed by merged PR #1972 |
| #67 Commit.Patch() slow | Needs a GitHub Enterprise fork setup to reproduce |
| #235 pre-receive hook errors | Environment-specific bug, not really perf |

**My read:** none of these was a tractable, self-contained win. The good ones were claimed; the unclaimed ones were roadmap-sized or unreproducible.

So instead of forcing a weak issue, I switched approach: **read the hot paths myself and look for algorithmic waste.** That turned out to be the better decision — the problem I found had no issue precisely *because* nobody had looked at it from this angle.

---

## 3. The finding: an N+1 hiding behind an underscore

`transport.UploadPack` — the server side of `git fetch`/`git clone` — calls:

```go
havesWithRef, err = revlist.ObjectsWithRef(st, wants, nil)
```

And `ObjectsWithRef` is:

```go
all := map[plumbing.Hash][]plumbing.Hash{}
for _, want := range wants {
	hashes, err := Objects(s, []plumbing.Hash{want}, haves) // a FULL graph walk, per want
	for _, h := range hashes {
		all[h] = append(all[h], want)
	}
}
```

One complete traversal of the object graph **per want**. With W refs requested, the shared history is walked W times.

**The decisive detail.** I grepped every reference to the result. There are exactly three in the entire repository:

```
upload_pack.go:112   var havesWithRef map[plumbing.Hash][]plumbing.Hash   // declare
upload_pack.go:134   havesWithRef, err = revlist.ObjectsWithRef(...)      // assign
upload_pack.go:185   _, ok := havesWithRef[hu]                            // key-only test
```

The `_` on line 185 is the whole finding. The map's **values** — the per-want attribution that the extra W−1 walks exist to produce — are never read anywhere. The function does W walks to build information that is immediately discarded.

**My read:** this is the textbook N+1 shape, but transplanted from databases into graph traversal. What made it invisible is that it is not visible *from inside* `revlist` at all. Reading `ObjectsWithRef` alone, the per-want loop looks essential — it is the only way to produce the documented return type. You only see the waste by reading the sole caller and noticing an underscore.

---

## 4. Checking prior art before writing anything

Before building anything I checked whether this was known or already attempted. This mattered more than usual, because `revlist` is a package the maintainers have actively optimised.

**Searched issues.** No open issue describes it. The closest is **#1691 — "transport.UploadPack runs out of memory and is very slow with large repos (linux.git)"**: pjbgf investigated, shipped PR #1698, and the reporter confirmed the OOM *persisted* after 176 minutes. It was auto-closed by the stale bot as `NOT_PLANNED` — **not fixed**.

**Checked four PRs specifically** (the last four were suggested to me as candidates worth verifying):

| PR | What it actually did | Fixes the N+1? |
| --- | --- | --- |
| #1776 mmap `PackScanner` | Faster pack *reading* (`storage/filesystem/mmap`) | No — different layer |
| #1758 context I/O allocations | `utils/ioutil` only | No — unrelated |
| #1947 "Faster send pack" | Rewrote `revlist`; **rewrote `ObjectsWithRef` itself** | **No — preserved the per-want loop** |
| #1973 "Object walk painting" | Rewrote the walk with commit painting, `object_walk.go` only | No — never touched `ObjectsWithRef` |

**The key observation.** #1947 and #1973 both optimised the *inside of a single walk*. The redundancy lives at the **call boundary**. Making one walk 2× faster still leaves the total W× too high when the walk runs once per ref and the results are thrown away.

That may explain #1973's own opening line: *"After some of the changes to #1947, the performance gains weren't realised."* A caller that multiplies the walk by the ref count would mask a walk-level improvement on exactly this path.

**Correction I had to make to myself.** My first instinct was that #1947 *introduced* the regression — its diff clearly adds the `for _, want := range wants` loop. I checked the pre-#1947 code before claiming that, and it was **also** per-want (it constructed a fresh `map[plumbing.Hash]bool{}` visited set inside the loop for every object). #1947 rewrote the body and faithfully preserved existing semantics. Blaming a maintainer's merged PR on the basis of a diff I had not traced back would have been both wrong and a poor way to open a conversation.

---

## 5. The fix

`UploadPack` needs only membership, so one walk over all wants suffices:

```go
objs, err := revlist.Objects(st, wants, nil)
if err != nil {
	return fmt.Errorf("getting objects: %w", err)
}

reachable = make(map[plumbing.Hash]struct{}, len(objs))
for _, h := range objs {
	reachable[h] = struct{}{}
}
```

**+220 / −5** across 3 files. Only the caller changes.

**Correctness argument.** Reachability from a set of roots is the union of reachability from each root, so `∪_w Objects(s,[w],nil) == Objects(s,wants,nil)`. I verified the key sets are byte-identical on a branching history: merge commits, nested trees, disjoint subgraphs, wants that are ancestors of other wants, and duplicate wants.

I was careful to bound this: the equivalence relies on `haves == nil`, which is the case at the only call site. With non-empty `haves` the exclusion boundaries could interact, and I explicitly declined to claim it there.

**Scope decision.** `ObjectsWithRef` is exported public API and becomes unused in-tree after this change. I left it untouched rather than removing it — removal is a breaking change requiring an RFC — and raised the leave/deprecate/remove question in the issue instead. The PR uses `Refs #2350` rather than `Fixes` so that question is not auto-closed when the perf fix merges.

---

## 6. Measurements, and a methodology error I made

### The numbers

Isolating just the changed call (300-commit history, `-benchtime 200x -count 5`):

| wants | current `ObjectsWithRef` | single walk + set |
| ---: | --- | --- |
| 1 | 1,063 µs · 1.20 MB · 9,070 allocs | **908 µs · 0.96 MB · 8,152 allocs** |
| 4 | 3,125 µs · 3.07 MB | 778 µs · 0.96 MB |
| 64 | 38,244 µs · 41.8 MB | 869 µs · 0.96 MB |
| 256 | 124,660 µs · 166.4 MB · 1,578,725 allocs | **861 µs · 0.96 MB · 7,874 allocs** |

End-to-end through the real `UploadPack`, where packfile encoding dominates at ~17 ms:

| wants | before | after | effect |
| ---: | --- | --- | --- |
| 1 | 18.36 ms · 7.14 MB | 17.81 ms · 6.97 MB | ~unchanged |
| 16 | 26.74 ms · 18.06 MB | 17.73 ms · 7.00 MB | 1.5× faster, 2.6× less memory |
| 64 | 55.63 ms · 50.51 MB | 18.58 ms · 6.90 MB | 3.0× faster, 7.3× less memory |
| 256 | 101.63 ms · 107.14 MB | 17.22 ms · 7.08 MB | **5.9× faster, 15.1× less memory** |

**The honest headline is ~5.9×, not 145×.** The isolated number is real but measures a component that is a fraction of a real fetch. Reporting the isolated figure as the headline would have been an overclaim that collapses the moment a maintainer profiles it. The shape matters more than the ratio anyway: before, serving cost grows with ref count; after, it is flat and tracks history size.

### Two mistakes worth recording

**1. A "regression" that wasn't.** An early end-to-end run showed the 1-want case **5% slower** with the fix — reproducible across repeated runs, ~2.9σ, and it survived a drift check. That is the most common case (single-branch clone), so it would have been a real problem.

But the allocation counts moved the *opposite* way: fewer allocs, less memory, yet slower wall time. That contradiction is what made me distrust it. I isolated the changed call and found it **1.17× faster at 1 want** — it avoids allocating a one-element slice per reachable object. So the change is strictly better at every want count, and the end-to-end delta was binary-layout/GC noise around a 17 ms encode. Later runs showed parity or better.

I still reported it in the issue and PR rather than quietly dropping it.

**2. A benchmark comparison that measured nothing.** Re-running before/after *after committing the fix*, I got a "before" that showed no scaling at all — flat ~17 ms across all want counts. Cause: `git stash push plumbing/transport/upload_pack.go` only stashes **uncommitted** changes, and the fix was already in a commit. Both runs measured the fixed code.

I caught it because flat numbers contradicted every earlier measurement. The correct method was `git checkout upstream/main -- <path>`. That has its own trap: it writes to the **index** as well as the working tree, which briefly made `git diff` display the entire fix as uncommitted. I reset the index before committing so the commit contained only what I intended.

**Lesson:** a benchmark result that contradicts your model is more likely a broken harness than a discovery. Both errors were caught by noticing an internal contradiction, not by a tool.

---

## 7. Process: issue first, then PR

go-git's `AI_POLICY.md` says to open an issue before a PR for non-trivial changes. But I had measured the repository's actual behaviour during contribution 1, and re-verified it here:

| channel | maintainer engagement |
| --- | --- |
| Issues (last 14 opened) | **1 of 14** got a reply |
| External PRs (last 12) | **12 of 12** got a maintainer review |

The clinching example: **#2249 — the issue contribution 1 fixed — got zero maintainer replies** and sat quiet for a month. The PR was merged in a day.

**My read:** go-git is a PR-first project. Waiting for a reply on an issue would mean waiting on a channel that answers ~7% of the time. So I posted the issue as the design record — satisfying the policy's actual purpose, which is that maintainers are not ambushed by unexplained changes — and opened the PR shortly after, referencing it. The PR is deliberately scoped to the uncontroversial half, with the API question left open in the issue.

---

## 8. Review and CI

### Copilot's automated review — three comments, all valid

1. **`out.String()` in the benchmark helper** copied the entire response, including all packfile bytes, on every iteration — adding allocations to a benchmark whose purpose was measuring `UploadPack`. This was substantive, because those numbers *are* the justification for the PR. Fixed to return `out.Len()`, then **re-measured from scratch**: 5.9× / 15.1×, so the claim survived.
2. **My comment described the wrong thing** — it said "Find common commits/objects", but the block only collects what the wants reach; the "common" decision happens later. I had inherited that wording from the original line and carried it forward without re-reading it. Reworded.
3. **Unclear doc comment** (`points w refs`) — spelled out, parameters renamed `n, w` → `commits, refs`.

**My read:** the first comment was a genuine methodology catch. Worth taking seriously rather than dismissing as bot noise.

### The CI failure that wasn't mine

First run: **16 pass, 1 fail** — `test (stable, macos-latest)`.

Rather than assume either way, I gathered evidence:

- `test (oldstable, macos-latest)` **passed** — same OS, different Go version.
- Ubuntu and Windows passed on **both** Go versions.
- **Git Compatibility passed** — the suite run against a real `git` binary, the most relevant check for an upload-pack change.
- The same job was failing on unrelated branches, including a pure Renovate dependency bump and `releases/v5.x`.
- The change contains no OS-specific code.

I could not read the job log directly — the token lacks the Actions `read` scope, so `gh run view --log` returns empty. Pushing the review fixes re-triggered CI and gave the decisive second sample: **`test (stable, macos-latest)` passed.** Flake confirmed.

Final: **17/17 green**, `mergeable=MERGEABLE`.

---

## 9. What I learned

- **Read the caller, not just the function.** The waste was invisible inside `revlist`; it was legible only from the call site, in a single `_`. Four rounds of profiling-driven optimisation on this package missed it for that reason.
- **Check prior art properly, then check your accusation.** I nearly claimed #1947 introduced the regression on the strength of its diff. Tracing to the pre-#1947 code showed the behaviour was long-standing.
- **Report the honest number.** The isolated benchmark said 145×; end-to-end said 5.9×. Leading with 145× would have been discounted the moment someone profiled it.
- **Distrust results that contradict your model.** Both of my errors — the phantom 5% regression and the no-op before/after — were caught by noticing an internal contradiction (fewer allocs but slower; flat numbers where scaling was proven).
- **`git stash` does not revert committed work.** Obvious in hindsight, and it silently produced a meaningless benchmark comparison.
- **Measure the project's real conventions, not just its stated ones.** The stated rule is "issue before PR"; the measured reality is that issues get ~7% replies and PRs get 100% review. Both facts mattered in deciding how to proceed.
- **Volunteer your weakest evidence.** Flagging the unexplained 5% and the bounded #1691 claim costs little and is what makes the rest of the numbers credible.

---

## 10. Next steps

- Await maintainer review on #2351; respond **personally**, as AI_POLICY requires.
- Offer the `releases/v5.x` backport if the maintainers want it.
