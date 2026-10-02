# Contribution 2 — go-git

**Target repository:** [go-git/go-git](https://github.com/go-git/go-git)
**Issue opened:** [#2350 — `plumbing/revlist: ObjectsWithRef walks the object graph once per want, but UploadPack only needs membership`](https://github.com/go-git/go-git/issues/2350)
**Pull request:** [#2351 — `plumbing: transport, Walk the object graph once per upload-pack request`](https://github.com/go-git/go-git/pull/2351)
**Date opened:** 29 Aug 2026
**Status:** **merged 1 Oct 2026** by `pjbgf` — merge commit `9079000b`, 3 commits, +228 / −5, CI 17/17 green
**Base commit:** `52f84ef3`
**Local working copy:** `~/Desktop/MyStuff/Programming/JOSA/oss-contrib/go-git`

> Everything under "Observed" was read directly off GitHub or the repository on **29 Aug 2026**, except section 8's review and merge material, which is from **30 Sep – 1 Oct 2026**. Anything marked *My read* is my own judgment, not a claim by the project.

---

## 1. Goal for this contribution

Contribution 1 was a correctness fix (an unbounded recursion). This time the goal was explicitly a **performance problem** — an algorithmic improvement, an N+1, or similar.

Two things had changed since contribution 1, both in my favour:

- **PR #2337 was merged one day after opening.** That promoted me out of GitHub's "first-time contributor" state, so CI now runs automatically instead of waiting for a maintainer to approve the workflow.
- I had a working repository, a validated toolchain, and a screening method that had already been corrected once.

---

## The fix

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

## Measurements

### The numbers

Isolating just the changed call (300-commit history, `-benchtime 200x -count 5`):

| wants | current `ObjectsWithRef` | single walk + set |
| ---: | --- | --- |
| 1 | 1,063 µs · 1.20 MB · 9,070 allocs | **908 µs · 0.96 MB · 8,152 allocs** |
| 4 | 3,125 µs · 3.07 MB | 778 µs · 0.96 MB |
| 64 | 38,244 µs · 41.8 MB | 869 µs · 0.96 MB |
| 256 | 124,660 µs · 166.4 MB · 1,578,725 allocs | **861 µs · 0.96 MB · 7,874 allocs** |

End-to-end through the real `UploadPack`, where packfile encoding dominates at ~17 ms:

| wants | before                | after              | effect                             |
| ----: | --------------------- | ------------------ | ---------------------------------- |
|     1 | 18.36 ms · 7.14 MB    | 17.81 ms · 6.97 MB | ~unchanged                         |
|    16 | 26.74 ms · 18.06 MB   | 17.73 ms · 7.00 MB | 1.5× faster, 2.6× less memory      |
|    64 | 55.63 ms · 50.51 MB   | 18.58 ms · 6.90 MB | 3.0× faster, 7.3× less memory      |
|   256 | 101.63 ms · 107.14 MB | 17.22 ms · 7.08 MB | **5.9× faster, 15.1× less memory** |

**The honest headline is ~5.9×, not 145×.** The isolated number is real but measures a component that is a fraction of a real fetch. Reporting the isolated figure as the headline would have been an overclaim that collapses the moment a maintainer profiles it. The shape matters more than the ratio anyway: before, serving cost grows with ref count; after, it is flat and tracks history size.

---

## Review and CI

### Copilot's automated review — three comments

1. **`out.String()` in the benchmark helper** copied the entire response, including all packfile bytes, on every iteration — adding allocations to a benchmark whose purpose was measuring `UploadPack`. This was substantive, because those numbers *are* the justification for the PR. Fixed to return `out.Len()`, then **re-measured from scratch**: 5.9× / 15.1×, so the claim survived.
2. **My comment described the wrong thing** — it said "Find common commits/objects", but the block only collects what the wants reach; the "common" decision happens later. I had inherited that wording from the original line and carried it forward without re-reading it. Reworded.
3. **Unclear doc comment** (`points w refs`) — spelled out, parameters renamed `n, w` → `commits, refs`.

**My read:** the first comment was a genuine methodology catch. Worth taking seriously rather than dismissing as bot noise.

### Maintainer review

**Observed:** on 30 Sep, pjbgf requested changes — *"There's a small nit below, apart from that LGTM"* — against the regression test:

> `MultiACK` emits `ACK <hash> continue` whether the membership lookup succeeds or fails. The test therefore passes even with an empty reachable set.

He was right, and it is the most useful review comment I have received. My test requested the `multi_ack` capability and asserted `ACK <mid> continue`. The server picks the status like this:

```go
_, ok := reachable[hu]
if multiAckDetailed {
	status = packp.ACKCommon
	if !ok {
		status = packp.ACKReady
	}
} else if multiAck {
	status = packp.ACKContinue   // ok is never consulted
}
```

Under `multi_ack` the status is `continue` regardless of the lookup. **The test I wrote to guard the reachability set never looked at the reachability set.**

Fixed in `77ed5a13`: request `multi_ack_detailed`, assert `ACK <have> common`, and add a `NotContains` on `ACK <have> ready` so the failure mode is explicit rather than implied.

**The thing I then got wrong.** My instinct was to verify the new assertion by reverting the optimisation and watching the test fail. It still passed. The reason is already in section 5: `ObjectsWithRef` unions the per-want walks, so its key set is *identical* to `Objects(wants)`. The change is behaviour-preserving by construction, so **no test can distinguish old from new** — that is the PR's entire claim.

So "revert the fix" is the wrong negative control here. The right one is to break the property the test exists to protect. Restricting the walk to `wants[:1]` makes the have unreachable through the first want:

| | `multi_ack` / `continue` (before) | `multi_ack_detailed` / `common` (after) |
| --- | --- | --- |
| walk restricted to `wants[:1]` | **passes** | **fails on both assertions** |

The mutant answers `ACK <have> ready`. That table is what I replied with, and I volunteered the behaviour-preserving point rather than letting him try reverting and find it proves nothing.

**My read:** I had been treating "the test passes with the fix" as sufficient. It is not. A test earns its place only if there is a specific mutation it catches — and for a pure optimisation, that mutation is not "undo the optimisation", it is "break the invariant the optimisation relies on."

### Merged

**Observed:** pjbgf approved and merged on **1 Oct 2026** (merge commit `9079000b`), closing with *"Thanks for working on this. 🙏"*

**33 days** from opening to merge — 32 of them waiting for the first human review, then one day to turn the fix around and merge. Contribution 1 merged in a single day; the difference was that this one needed a maintainer with context on the negotiation protocol, not just on the bug.


---

## Outcome and what is left

**Merged.** Second contribution to go-git, after #2337. Serving an upload-pack fetch no longer scales with the number of refs requested: at 256 wants it went from 101.6 ms / 107.1 MB to 17.2 ms / 7.1 MB, and the cost is now flat in ref count and tracks history size instead.
