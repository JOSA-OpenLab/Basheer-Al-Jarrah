
**Task:** Find untested code in an active GitHub repo, write a well-structured Arrange/Act/Assert test file, and submit a PR.

**Repo:** boltons — https://github.com/mahmoud/boltons A widely-used pure-Python utility collection, actively maintained (latest release 26.0.0, June 2026).

**PR:** https://github.com/mahmoud/boltons/pull/414

**File added:** `tests/test_pathutils.py`

## What I added

A dedicated test file for `boltons/pathutils.py`, covering its three functions with Arrange/Act/Assert tests:

- `augpath` — directory replacement and preservation; missing, empty, and replaced extensions; `multidot` true/false for both suffix and extension; the full prefix/base/suffix/ext combination.
- `shrinkuser` — exact home to `~`, home-prefixed subpath, the no-separator guard (a sibling like `~extra` must not be shrunk), `normpath` collapsing of `//` and `..`, and the custom home symbol on both the exact-home and subpath branches.
- `expandpath` — tilde expansion, environment variable, combined tilde + variable, undefined variable passthrough, and a plain path left unchanged.

No source changes — tests only.

## Why

`pathutils` was the only major `*utils` module in the repo without a dedicated test file. Two reasons it was worth filling:

1. The module's doctests cover the headline examples, but several parameter combinations and edge cases were exercised nowhere — the `shrinkuser` no-separator guard, empty/missing extensions in `augpath`, custom home symbol on the exact-home branch, undefined-variable handling in `expandpath`.
2. `--doctest-modules` is only applied by the tox command, not in `pyproject.toml`, so a plain `pytest` run collects no doctests at all. The unit tests give the module coverage independent of how the suite is run.

## How I worked

- Scouted by mapping every `boltons/*.py` source module against its `tests/test_*.py`. Six modules had no test file; I picked `pathutils` because its functions are pure and deterministic — clean to test without fixtures.
- Probed each function's real runtime behavior before writing assertions, so the expected values match actual output rather than assumptions.
- Made every home-directory assertion derive from `expanduser('~')` and built paths with `os.path.join`, so the suite is machine- and platform-independent — relevant given the repo's Windows/macOS CI matrix.
- Used the existing house style: plain functions with `assert`, matching the other `test_*utils.py` files; `monkeypatch` for the env-var test.

## Verification

- `pytest tests/test_pathutils.py` → all unit tests pass.
- `pytest --doctest-modules boltons/pathutils.py tests/test_pathutils.py` → passes, confirming the new file coexists with the project's CI doctest mode.
