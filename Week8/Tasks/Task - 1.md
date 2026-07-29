# Journal — Profile & Fix: boltons IndexedSet

## Goal

Profile a real workflow in a project I use, find a >50ms bottleneck with a
flame graph, report upstream with numbers, fix if safe. Target: `boltons` —
I have a merged PR there already (#414, pathutils tests) and use it in scripts.

## Workflow profiled

Bulk removal from `setutils.IndexedSet` via `difference_update()` — removing
half of an n-element set (scattered pattern). Baseline at n=160,000:
1,197.9 ms, 863x slower than builtin `set`, and the gap grows with n.
Doubling n roughly quadruples the time: quadratic.

## Method

1. Scaling probe at 5 sizes (10k → 160k) to establish the complexity class
2. py-spy 0.4.2 @ 250 Hz → flame graph: the whole tower is
   `difference_update → remove → _cull → _compact` (45% self-time on the
   hottest line inside `_compact` alone)
3. Cross-checked with cProfile: `_compact` is 91% of cumulative time with
   207 calls for 80,000 removals — matching the predicted 80,000/384 ≈ 208,
   which is what turned the hypothesis into a proven mechanism
4. Read the source (`remove`, `_cull`, `_compact`, `_get_real_index`) before
   touching anything

## Finding

`remove()` records a dead interval per scattered removal; once intervals exceed
a hardcoded 384, `_cull()` runs `_compact()` — a full O(n) rewrite of the
backing list. Bulk removal of k items = O(n·k/384). The cap is intentional: it
bounds the linear interval scan in `_get_real_index()` so indexing stays fast.
Fair tradeoff for single removals — but the bulk update methods know the whole
removal set up front and can rebuild once in O(n).

## Fix

`_bulk_discard()`: one-pass rebuild of `item_list` + `item_index_map`, used
by `difference_update`/`intersection_update` when the removal count exceeds
the threshold. Small removals keep the old path (zero behavior change).
`symmetric_difference_update` untouched: its single-pass loop has
order-dependent toggle semantics when the argument contains duplicates, which
a bulk path can't reproduce. ~30-line diff + 3 tests.

## Before / After (i7-12700H, best of 3)

| n | before | after | speedup |
|---|---|---|---|
| 10,000 | 7.8 ms | 1.7 ms | 4.6x |
| 40,000 | 86.1 ms | 7.7 ms | 11x |
| 160,000 | 1,197.9 ms | 31.2 ms | 38x |

Scaling class: quadratic → linear. Test suite: 9 → 12 passed (3 new).

## Upstream

- Issue: https://github.com/mahmoud/boltons/issues/439 (flame graph attached)
- PR: https://github.com/mahmoud/boltons/pull/440
- Status as of 2026-07-29: open, awaiting maintainer review
