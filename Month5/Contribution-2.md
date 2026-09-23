# Contribution 2 — boltons

**Repo:** [mahmoud/boltons](https://github.com/mahmoud/boltons) · **PR:** [#515](https://github.com/mahmoud/boltons/pull/515)
**Opened:** 23 Sep 2026 · **Status:** open · Second PR here, after [#440](https://github.com/mahmoud/boltons/pull/440) merged in July

---

## What I did

Made `soft_sorted` in `iterutils` linear in the size of its `first`/`last` priority lists. It was O(n·f) — sorting 4000 items against a 4000-entry priority list took 100 ms, and now takes 0.89 ms.

## Why

`soft_sorted` floats some elements to the top and sinks others to the bottom, sorting everything else normally. Three things were stacked in it:

```python
other = [x for x in seq if not ((first and key(x) in first) or (last and key(x) in last))]
first = sorted([x for x in seq if key(x) in first], key=lambda x: first.index(key(x)))
```

`key(x) in first` is a linear scan of `first`, run once per item. `first.index(...)` is a second scan. And `key()` gets recomputed three to five times per item — 248 calls for 50 items in one of my tests.

None of that matters when `first` has five entries. It matters a lot when the priority list scales with the input, and the growth was about 4× per doubling — the textbook quadratic shape.

## How

Index `first` and `last` by first occurrence in a dict, and evaluate `key()` once per item by decorating each one with its key up front.

The interesting part was what *couldn't* be indexed. Two cases would have broken:

- **`first` given as a string.** The docstring shows `first='za1'`, and `in` on a string tests for a *substring*, not an element. A dict of characters would answer differently for multi-character keys.
- **Unhashable keys.** The old code only ever needed equality, never hashing, so callers could pass lists or dicts as keys.

Both fall back to the original scan. That's most of the added code, and it's why the diff is +125 rather than +20.

I also broke it twice before getting it right. My first version decorated every item even when there was no `first` or `last` at all, which made the most common call **4–5× slower**; a short-circuit turned that into a 1.7× win. The second problem was using a Python lambda for the bulk sort where `operator.itemgetter` does the same job in C.

**Verifying it.** I ran the new implementation against a verbatim copy of the old one over 201,112 cases — exhaustive over small integer sequences with every combination of `first`, `last` and `reverse`, plus the doctests and randomised string cases. Identical on all of them, including the awkward semantics I had to preserve: duplicates in `first` (where `.index` returns the earliest position) and an element listed in *both* `first` and `last`, which the original emits at both ends.

`soft_sorted` had no tests beyond its doctests, so I added six. One asserts `key()` is called once per item and fails on the old code; the other five lock in the semantics and pass either way, since the change is meant to preserve them. Full suite: 678 passed.

## Numbers

| case | before | after | |
| --- | ---: | ---: | --- |
| n = first = 1000 | 6.53 ms | 0.19 ms | 35× |
| n = first = 4000 | 100.02 ms | 0.89 ms | 112× |
| n=5000, first=10, costly key | 1.54 ms | 1.15 ms | 1.33× |
| n=5000, no first/last | 0.20 ms | 0.12 ms | 1.71× |
| n=5000, first=10, cheap key | 0.92 ms | 0.97 ms | 0.95× |

That last row is the cost, and it's in the PR: with a tiny priority list and a cheap key, decorating each item is about 5% slower than just recomputing it. I left it visible rather than dropping the row — it's the one number that argues against the change, and a maintainer will find it anyway.
