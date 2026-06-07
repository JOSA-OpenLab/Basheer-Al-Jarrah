# git bisect Journal

## Repository Information

- **Repository:** `more-itertools/more-itertools`
- **URL:** https://github.com/more-itertools/more-itertools
- **Language / build:** Pure Python (no compile step)
## Behavior Change Description

**Function:** `more_itertools.windowed(seq, n, ...)` with window size `n = 0`.

| Code                           | Old behavior (working)                   | New behavior (changed)             |
| ------------------------------ | ---------------------------------------- | ---------------------------------- |
| `list(windowed([1, 2, 3], 0))` | returns `[()]` (yields one empty window) | raises `ValueError: n must be > 0` |

This is the change listed in the 11.0.0 release notes: *"`windowed` now raises `ValueError` when given a window size of `0`."*

**How it is tested (one line, deterministic):**
```python
import more_itertools as mi
list(mi.windowed([1, 2, 3], 0))   # old: [()]   new: raises ValueError
```

**Test oracle used for bisect** (`bisect_test.py`):
```python
import sys
try:
    import more_itertools as mi
except Exception:
    sys.exit(125)              # cannot import -> skip this commit
try:
    list(mi.windowed([1, 2, 3], 0))
except ValueError:
    sys.exit(1)                # NEW behavior  -> "bad"
except Exception:
    sys.exit(125)              # unexpected    -> skip
sys.exit(0)                    # OLD behavior  -> "good"
```
Exit-code contract for `git bisect run`: `0` = good, `1` = bad, `125` = skip.

## Setup Instructions  (EndeavourOS / zsh)

```zsh
# 1. Clone the repository (SSH)
git clone git@github.com:more-itertools/more-itertools.git
cd more-itertools

# 2. Save the test oracle OUTSIDE the repo so git checkouts never disturb it
cat > ../bisect_test.py <<'PY'
import sys
try:
    import more_itertools as mi
except Exception:
    sys.exit(125)
try:
    list(mi.windowed([1, 2, 3], 0))
except ValueError:
    sys.exit(1)
except Exception:
    sys.exit(125)
sys.exit(0)
PY

# 3. Make the checked-out working tree importable, and don't litter __pycache__
export PYTHONPATH="$PWD"
export PYTHONDONTWRITEBYTECODE=1
```

> Gotcha worth recording: if you instead run `python /some/other/dir/test.py`, Python puts *that script's* directory on `sys.path` (not the repo), so `import more_itertools` fails and **every** commit gets skipped. Setting `PYTHONPATH="$PWD"` (repo root) fixes it. For a heavier but bullet-proof alternative, `pip install -e .` at each step via `git bisect run sh -c 'pip install -e . -q && python -c "..."'`.

## Exact Git Commands

```zsh
# endpoints (verified real hashes)
#   good = v10.8.0 -> 8c1a6ef241b51ff055e89219f050ccf4f15f37f6   (windowed(...,0) == [()])
#   bad  = v11.0.0 -> 5b43c0dc5f67713130ab534d448f8db1cf493e09   (raises ValueError)

git bisect start
git bisect bad  v11.0.0
git bisect good v10.8.0

# automated search: runs the oracle at each step, marks good/bad by exit code
git bisect run python3 -B ../bisect_test.py

# after it reports the first bad commit:
git bisect log        # full decision log
git bisect reset      # restore your branch
```

## Bisect Walkthrough

Endpoints span **153 commits** (`v10.8.0..v11.0.0`); `git bisect` converged in **7 probes** (`76 → 37 → 18 → 9 → 4 → 2 → 0` revisions remaining). Each probe ran `python3 -B ../bisect_test.py`; "good" = `windowed([1,2,3],0)` returned `[()]`, "bad" = it raised `ValueError`.

| # | Commit (short) | Title | Observed | Mark |
|---|---|---|---|---|
| — | `5b43c0dc5f67` | v11.0.0 (Merge PR #1136) | raises `ValueError` | **bad** (start) |
| — | `8c1a6ef241b5` | v10.8.0 (Merge PR #1071) | returns `[()]` | **good** (start) |
| 1 | `ca8c40333a50` | Merge PR #1110 (issue‑1109‑mypy) | returns `[()]` | good |
| 2 | `1c21c3ae9c79` | Merge PR #1127 (running‑mean‑recipe) | returns `[()]` | good |
| 3 | `93c1cf8484f0` | Revert "first_true: pred → predicate" | returns `[()]` | good |
| 4 | `eaaa55d531cd` | Merge PR #1138 (sync_recipe_parameters) | returns `[()]` | good |
| 5 | `06626d5d47dc` | "callback_iter is deprecated" | raises `ValueError` | bad |
| 6 | `a6a2edcc9d9d` | Merge PR #1139 (major_wishlist_1057) | raises `ValueError` | bad |
| 7 | `71b46b06fb48` | **Issue #1057: Raise exception for invalid size in windowed()** | raises `ValueError` | **bad → first bad** |

`git bisect` output ended with:
```
71b46b06fb48abcd2f7a26d74c148a650d340386 is the first bad commit
    Issue #1057: Raise exception for invalid size in windowed()
 more_itertools/more.py | 7 ++-----
 tests/test_more.py     | 7 ++++---
```

## Culprit Commit Analysis

- **First bad commit (culprit):** `71b46b06fb48abcd2f7a26d74c148a650d340386`
- **URL:** https://github.com/more-itertools/more-itertools/commit/71b46b06fb48abcd2f7a26d74c148a650d340386
- **Title:** *Issue #1057: Raise exception for invalid size in windowed()*
- **Author / date:** Raymond Hettinger — 2026‑03‑31
- **Merged via:** PR #1139 (merge commit `a6a2edcc`), referencing issue **#1057**
- **Last good commit (its parent):** `e4d2a4a2a97246a73856754b2c4866d7f41d4875` ("Merge pull request #1126 …")

**Why bisect identified it:** It is the earliest commit on the path from `v10.8.0` to `v11.0.0` whose parent still returns `[()]` while it raises `ValueError`. Verified directly: parent `e4d2a4a2` → exit `0` (good), culprit `71b46b06` → exit `1` (bad). It is a descendant of `v10.8.0` and an ancestor of `v11.0.0`.

**What changed** in `more_itertools/more.py`, inside `windowed()`:
```diff
-    if n < 0:
-        raise ValueError('n must be >= 0')
-    if n == 0:
-        yield ()
-        return
+    if n <= 0:
+        raise ValueError('n must be > 0')
     if step < 1:
         raise ValueError('step must be >= 1')
```
The old code special-cased `n == 0` by yielding a single empty tuple and returning; the new code folds `n == 0` into the `n <= 0` guard and raises `ValueError('n must be > 0')`. That is exactly the observed behavior flip.

## Conclusion

`git bisect` correctly and unambiguously isolated the single commit that replaced the `n == 0 → yield ()` special case with a `ValueError`. The result was independently confirmed by testing the culprit and its parent directly, and the source diff matches the observed behavior change exactly.

### Verified links
- Repo: https://github.com/more-itertools/more-itertools
- Culprit commit: https://github.com/more-itertools/more-itertools/commit/71b46b06fb48abcd2f7a26d74c148a650d340386
- Parent (last good): https://github.com/more-itertools/more-itertools/commit/e4d2a4a2a97246a73856754b2c4866d7f41d4875
- Tag v10.8.0: https://github.com/more-itertools/more-itertools/releases/tag/v10.8.0
- Tag v11.0.0: https://github.com/more-itertools/more-itertools/releases/tag/v11.0.0
- Issue #1057: https://github.com/more-itertools/more-itertools/issues/1057
- PR #1139: https://github.com/more-itertools/more-itertools/pull/1139
