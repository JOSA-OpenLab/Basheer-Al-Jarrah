# Contribution 3 — SQLGlot

**Repo:** [tobymao/sqlglot](https://github.com/tobymao/sqlglot) · **PR:** [#8844](https://github.com/tobymao/sqlglot/pull/8844)
**Opened:** 30 Oct 2026 · **Merged:** 30 Oct 2026 by `georgesittas` (`6f90b498`) — 1 commit, 2 files, +51 / −5

---

## The contribution

`sqlglot.diff()` returned its edits in an order that changed between processes. `_generate_edit_script` iterated two `set`s of `id()` values and a dict built from a set of `(id, id)` tuples — and `id()` is an object address, so the sequence of `Remove`/`Insert`/`Keep`/`Update` edits was shuffled differently on every run for identical input.

The fix iterates `_source_index` and `_target_index` instead, which are already dicts built from `bfs()`, giving stable tree order at no extra cost. Grouping is unchanged and the set of edits produced is unchanged — only their order becomes stable. Ships with a regression test that fails on `main`.

## Goal

Make `diff()` output comparable. Anything snapshotting or asserting on an edit script previously had to sort it first, and the differ is deterministic everywhere else — `_compute_matching_set` already avoided this exact trap by walking `bfs()` rather than iterating the id sets.

## Review

Approved and merged as submitted, no changes requested — *"This seems fine, thanks for the PR"*. CI green across 3.9–3.15.

## Issues

None. Found while benchmarking [#8838](https://github.com/tobymao/sqlglot/pull/8838) — a differential check reported mismatched hashes, and checking the baseline showed the instability was pre-existing rather than caused by that change. Kept as a separate PR so a correctness fix wasn't bundled into a performance diff.

## Outcome

Merged into `main` the same day it was opened, unmodified.
