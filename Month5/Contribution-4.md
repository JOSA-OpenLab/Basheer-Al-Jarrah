# Contribution 4 — celery

**Repo:** [celery/celery](https://github.com/celery/celery) · **PR:** [#10840](https://github.com/celery/celery/pull/10840)
**Opened:** 11 Oct 2026 · **Status:** open, waiting on the first-time-contributor CI gate

---

## What I did

Made `chain()` construction linear instead of quadratic. Building a chain of 800 tasks went from 631 ms to 3.7 ms.

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
| 100 | 10.60 ms | 0.53 ms | 20× |
| 200 | 39.95 ms | 0.96 ms | 42× |
| 400 | 154.41 ms | 1.88 ms | 82× |
| 800 | 631.38 ms | 3.72 ms | 170× |

Before quadruples per doubling, after doubles. 606 tests pass across canvas, chord, result, tasks and builtins.

## What I got wrong

Four things, and all four were caught by checking rather than by reading.

- **My first implementation failed 3 canvas tests.** `chain(group(...), a, b, c)` has to collapse into a chord, and I was ending the fold when the accumulator stopped being a chain instead of continuing with it.
- **My first equivalence test rested on a false premise.** I assumed `chain(X, Y)` always equals `X | Y`, because the docstring says so. It doesn't — four of my thirteen shapes already differ on current main. I only found out by running the same check against unpatched code.
- **My first A/B measurement was meaningless.** I loaded the original `canvas.py` as a second module to compare against, which breaks `isinstance` dispatch, so the "before" column wasn't running the real code. It showed a 1× speedup, which is what made me look.
- **I ran the full unit suite in one process and the kernel killed the session for memory.** Ran targeted modules in small batches after that.

The through-line: my own 24-shape differential test passed happily while the implementation was wrong. Celery's suite caught it in one run.

## Scope

Construction cost only. The serialized message growth that's the other half of #9202 is being handled separately, so the PR says `Refs` rather than `Fixes`.
