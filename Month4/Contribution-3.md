# Contribution 3 — more-itertools

**Repo:** [more-itertools/more-itertools](https://github.com/more-itertools/more-itertools) · **Issue:** [#1017](https://github.com/more-itertools/more-itertools/issues/1017) · **PR:** [#1265](https://github.com/more-itertools/more-itertools/pull/1265)
**Opened:** 8 Sep 2026 · **Closed:** 10 Sep 2026 by `bbayles`, unmerged — declined on design grounds, not folded

>[!note] The PR was closed by the maintainer based on his design preferences. I still believe the changes are valuable and worth considering for a future merge, though.
---

## What I actually shipped

more-itertools is a Python library with about 4,000 stars and, crucially, **eleven open issues**. That's the whole reason this one worked. Every Go repo I'd been screening had hundreds, and a `good first issue` label on a popular repo is basically a magnet. A small, well-tended tracker turned out to be a much better place to look, which is the opposite of what I assumed going in.

`sliding_window` picks a strategy based on window size: above 20 it uses a `deque`, between 3 and 20 a `tee`/`zip` trick, and special cases below that. Issue #1017 from June 2025 asked whether the cutover at 20 was right and whether a plain list would beat the deque. A maintainer replied that he was inclined to swap the deque out. Then it sat for fifteen months and nobody did it.

I benchmarked all three strategies on CPython 3.14 over a 100,000-item stream, and the result was cleaner than the issue suggested: **the deque is the slowest of the three at every size I tested** — and it's exactly where every window above 20 gets sent.

The reason is the interesting bit. A `deque` with `maxlen` appends and evicts in O(1), while `list.append` plus `del list[0]` is the textbook O(n) antipattern. On complexity alone the deque is obviously right, and that's presumably why it's been there for years. But the window gets converted to a tuple on *every single iteration*, and CPython has a fast path for list→tuple and none for deque→tuple. That conversion dominates the loop, and it more than pays for the O(n) delete.

Asymptotics point one way, constant factors point the other, and here the constants win by roughly 3×. You only see it if you measure.

The change itself:

```python
def _sliding_window_list(iterable, n):
    iterator = iter(iterable)
    window = list(islice(iterator, n - 1))
    for x in iterator:
        window.append(x)
        yield tuple(window)
        del window[0]
```

Measured through the public API, before and after:

| n | before | after | |
| ---: | ---: | ---: | --- |
| 20 | 9.20 ms | 8.83 ms | unchanged, same path |
| 21 | 20.63 ms | 8.24 ms | 2.5× |
| 100 | 66.12 ms | 18.93 ms | 3.5× |
| 800 | 342.37 ms | 113.25 ms | 3.0× |

Nothing at or below 20 changes — it takes the same route it always did.

To be clear about credit: the list variant and the explanation for why it wins are **@pochmann3's**, from the issue thread. What I added is fresh benchmarks on a current Python, a correctness proof, and the implementation. That's in the PR and in my comment, because it's plainly visible to anyone reading the thread and pretending otherwise would be both wrong and stupid.

## Checking it

I diffed the old and new implementations across 2,372 combinations of window size and input length — plain sequences, one-shot generators, and unhashable payloads — and the output is identical. The full suite passes (909 tests, 5 skipped), 145 doctests pass, and both `ruff format --check` and `ruff check` are clean. I also renamed a test called `test_deque_version`, since there's no deque in that path anymore.

I deliberately did **not** move the `n > 20` cutover, even though my numbers suggest the list also beats `tee`/`zip` from around n=8. The maintainer only signalled on the deque swap; the threshold is a separate and riskier call, and it belongs in the issue rather than bundled into this PR.

## What I told them up front

Two caveats went into the PR rather than waiting for review to find them.

The benchmarks are CPython 3.14 only, and the entire argument rests on a CPython implementation detail. Their CI also runs PyPy, which I couldn't test locally, so I said so and offered to hold or adjust. That invites a maintainer to pause the PR, which is a genuine cost — but PyPy runs in their CI regardless, and being the one who raised it reads a lot better than being caught by it.

The other was the cutover decision above, with my reasoning for keeping it separate.

I posted the benchmark table on the issue first and then opened the PR referencing it, since the issue was where the maintainer had already engaged.

## Where it stands

CI is queued pending maintainer approval, which is normal for a first PR to this repo — two other open PRs are sitting in exactly the same state. Once it runs I need to confirm the whole matrix is green, PyPy included. If PyPy regresses, the honest options are a version-conditional dispatch or pulling the change; I wouldn't argue to keep it on CPython numbers alone.

If the maintainers want the cutover looked at too, that's a follow-up in #1017.

## Outcome

**Closed unmerged** on 10 Sep 2026 by `bbayles`. He first closed it accidentally on 8 Sep (*"Apologies; I mixed this PR up with another. Will review soon!"*), reopened, then closed it deliberately two days later:

> I think Raymond's comment here remains accurate: [#1017 (comment)](https://github.com/more-itertools/more-itertools/issues/1017#issuecomment-3002043652) So we'll pass on this PR. Thanks for the contribution

The reasoning is Raymond Hettinger's, from the issue, and it is not about the measurements — he accepts that the list variant is faster. He rejects *why* it is faster:

> ISTM the sliding window list variant relies on a fragile implementation detail that could easily change as the interpreter shifts to no-gil logic. Personally, I would stick with the deque variant despite the timings. A deque with maxlen is specifically designed for the sliding window use case. […] The *only* slow part of `_sliding_window_deque` is the deque-to-tuple conversion: `yield tuple(window)`. The deque itself is optimal. It only lacks a `PyDeque_AsTuple` short-cut.

So the win came from `PyList_AsTuple` bypassing the iterator protocol, which `PyDeque_AsTuple` has no equivalent for — a CPython implementation detail rather than an algorithmic improvement. The maintainers would rather carry the slower-but-principled `deque` than a faster version resting on something free-threading could invalidate.

**Nothing of mine shipped**, and `sliding_window` still uses the deque. The decline predates my PR: Raymond's comment is from June 2025, on the issue I was working from. The lesson isn't about the benchmark — it's that a performance PR against a tracked issue needs the issue's *existing* discussion read to the end first. Had I read that comment before writing the patch, I'd have known the answer was already no, and the right move would have been arguing the point in the issue rather than submitting code.