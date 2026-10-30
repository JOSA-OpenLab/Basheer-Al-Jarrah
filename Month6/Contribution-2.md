# Contribution 2 — SQLGlot

**Repo:** [tobymao/sqlglot](https://github.com/tobymao/sqlglot) · **PR:** [#8838](https://github.com/tobymao/sqlglot/pull/8838)
**Opened:** 30 Oct 2026 · **Merged:** 30 Oct 2026 by `georgesittas` (`2ef935d6`) — 7 commits, 1 file, +28 / −10

---

## The contribution

`ChangeDistiller._compute_matching_set` is the core of SQLGlot's semantic SQL differ: it compares every unmatched source node against every unmatched target node. For each pair it recomputed both nodes' leaf sets and scanned the entire leaf matching set to count their common leaves — none of which depends on the pair.

The PR indexes the matching set by source leaf once up front, computes each node's leaf set once instead of per pair, and bounds the per-pair scan to the leaves under the current source node.

## Goal

Remove the `O(S*T*L)` term from `diff()`. `_compute_matching_set` was 95% of runtime on an 1172-node pair. Diffing two 2212-node queries went from 550 ms to 390 ms, with the speedup growing as queries get larger.

## Review

One round, `CHANGES_REQUESTED`, merged about 35 minutes later. Three changes:

- **Lazy source leaves (required).** Hoisting the source leaf set to the outer loop meant walking a node's subtree even when no target shared its type — the original only did that work after the `_is_same_type` check passed. Reworked to compute it on first type match.
- **Comments removed.** The maintainer stripped the explanatory comments above each change.
- **His own follow-up commit** (`475f18c5`) evicting a target's cached leaf set once it is matched, since the cache otherwise grew to every target ever compared and never shrank.

The structure of the change survived; what moved was when the source leaves get computed, plus the cache cleanup.

## Issues

None — no open issue existed; the problem was found by profiling.

## Outcome

Merged into `main` the same day it was opened.
