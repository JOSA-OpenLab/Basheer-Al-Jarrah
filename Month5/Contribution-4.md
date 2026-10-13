# Contribution 4 — celery

**Repo:** [celery/celery](https://github.com/celery/celery) · **PR:** [#10840](https://github.com/celery/celery/pull/10840)
**Opened:** 11 Oct 2026 · **Merged:** 13 Oct 2026 by `auvipy` (`e39d0313`) — 11 commits, +256 / −3

---

## What I did

Made `chain()` construction linear instead of quadratic. Building a chain of 800 tasks went from 655 ms to 5.6 ms.

## Why

Celery is the standard Python task queue, and `chain` is one of its core canvas primitives — the thing you use to say "run these tasks in order". Constructing one shouldn't cost anything much.

But `chain.__new__` builds the chain by folding the tasks with the `|` operator:

```python
new_instance = reduce(operator.or_, tasks, _chain())
```

and every `_chain.__or__` runs `unchain_tasks()`, which is `[t.clone() for t in self.tasks]`. It re-clones every task the chain already holds, on every single step. Folding *n* signatures therefore does *n(n−1)/2* clones.

The profile matched the formula exactly: for 800 tasks, `Signature.clone` was called **319,600** times — precisely 800·799/2 — and cloning was **96%** of construction time.

This isn't theoretical. Issue [#9202](https://github.com/celery/celery/issues/9202) has been open since August 2024 reporting 100% CPU and memory on chains past 150 tasks, and a maintainer asked on it: *"How do you propose we fix this?"*

## How

The composition rules — empty groups are skipped, a group of one unrolls, a group followed by a task becomes a chord, a task after a chord extends the chord body, consecutive chords collapse — only ever look at the **last** step of the chain. So they can be applied to a list in place instead of rebuilding the whole chain each time. One pass, each task cloned once.

The part that needed care was what the single pass *can't* express. An empty chain followed by a group yields the group, not a chain, and from there the remaining tasks compose through the group's own operator. I hand those cases back to `|` rather than trying to reimplement them.

**One behaviour change, which I put in the PR.** The old fold leaves the last task of the finished chain aliased to the caller's signature, because the final step clones the accumulated tasks but not the incoming one. Every other position is a clone. Mine clones that one too. I chose consistency over bit-for-bit fidelity, since reproducing the asymmetry means leaving a caller-owned object where later composition can mutate it in place — and offered to restore the old behaviour if the maintainers prefer it.

**Tests.** Two. One asserts the clone count stays linear and **fails on current main** (a 40-task chain clones 780 times there). The other pins the group/chord/nested-chain/empty-group outcomes and passes both before and after — it documents what the rewrite had to preserve.

## Numbers

Patch applied, then reverted with `git checkout HEAD` on the same checkout.

| tasks | before | after | |
| --- | ---: | ---: | --- |
| 100 | 10.65 ms | 0.75 ms | 14× |
| 400 | 163.19 ms | 2.79 ms | 58× |
| 800 | 655.20 ms | 5.62 ms | 117× |
| 1600 | — | 11.36 ms | |

Before quadruples per doubling, after doubles. Final figures, after the review changes below.

## Review

Four rounds, three of which changed the code.

- **MehrazRumman — a dropped task.** When a nested chain began with a group, the steps after it were silently discarded: `chain(_chain(group(a, b), add.s(10)), add.s(20))` lost `add(10)`. My fold returned early instead of composing the rest onto the new result, which is what `_chain.__or__` does. Fixed, plus three regression tests that fail without the fix.
- **MehrazRumman — a behaviour change.** `|` never clones its right-hand side, so the last task of a chain is the caller's own object. My version cloned it, which would break `c.link(...)` after the fact. Threaded a flag through so the final step is passed on uncloned; no behaviour change left in the PR.
- **auvipy — coverage.** Of the nine uncovered lines, five were dead code I had copied over from `__or__` (a chord-collapse rule that cannot trigger in a single pass — 0 hits in 20,000 canvases, and removing it changed nothing in the differential). The other four were real gaps, now covered.
- **auvipy — integration tests.** Added two to `t/integration/test_canvas.py` and ran them against a redis broker. Declined to add a smoke test I could not execute locally and said so.
- **GangEunzzang — shared errbacks.** `unchain_tasks()` hands the same errback object to every task; the old fold separated those copies by re-cloning each step, mine left them shared, so a per-signature stamping visitor wrote two steps' stamps into one errback. Cloning the non-final steps at the end fixed it. Over 3,000 stamped canvases: main 154, mine 192 before and 154 after — parity, not just close.

## What I got wrong

Four things, and all four were caught by checking rather than by reading.

- **My first implementation failed 3 canvas tests.** `chain(group(...), a, b, c)` has to collapse into a chord, and I was ending the fold when the accumulator stopped being a chain instead of continuing with it.
- **My first equivalence test rested on a false premise.** I assumed `chain(X, Y)` always equals `X | Y`, because the docstring says so. It doesn't — four of my thirteen shapes already differ on current main. I only found out by running the same check against unpatched code.
- **My first A/B measurement was meaningless.** I loaded the original `canvas.py` as a second module to compare against, which breaks `isinstance` dispatch, so the "before" column wasn't running the real code. It showed a 1× speedup, which is what made me look.
- **I ran the full unit suite in one process and the kernel killed the session for memory.** Ran targeted modules in small batches after that.

The through-line: my own 24-shape differential test passed happily while the implementation was wrong. Celery's suite caught it in one run.

## Scope

Construction cost only. The serialized message growth that's the other half of #9202 was handled separately (#10743, #10824), so the PR says `Refs` rather than `Fixes` and the issue stays open.
