# Contribution 4 — SQLGlot

**Repo:** [tobymao/sqlglot](https://github.com/tobymao/sqlglot) · **PR:** [#8924](https://github.com/tobymao/sqlglot/pull/8924)
**Opened:** 6 Nov 2026 · **Status:** open, mergeable, CI green (3.9–3.15 all pass), awaiting review — 1 commit, 7 files, +35 / −0

---

## The contribution

Dialects whose tokenizers delimit identifiers with brackets cannot express a slice inside one. SQLGlot emitted `x[1:2]` for them anyway, producing SQL that parses back as an **alias** rather than a range:

```
tsql:    x[1:2]  ->  x AS [1:2]
sqlite:  x[1:2]  ->  x AS "1:2"
exasol:  x[:]    ->  x AS ":"
```

Silently — a slice transpiled into one of those dialects became a column alias with nothing logged.

Adds a `SUPPORTS_ARRAY_SLICING` generator flag (following the existing `SUPPORTS_WINDOW_EXCLUDE` pattern), default `True`, set `False` for `tsql` (and `fabric`, which inherits it), `sqlite`, `tableau`, `exasol` and `dax`. The generated SQL is unchanged; only the diagnostic is added, so no existing expectation moves.

## Goal

Turn silent semantic corruption into a warning, or an error under `ErrorLevel.RAISE`. Found by sweeping `tests/fixtures/identity.sql` across all 33 dialects for a property the suite doesn't check — that generated SQL re-parses to itself — then filtering for *structural* change (an `Alias` appearing, a `Slice` vanishing) rather than intentional normalisation like `INT(x)` → `CAST(x AS INT)`.

The dialect list is derived from a measured property rather than a guess about which engines support slicing: all 6 bracket-quoting dialects lose slices 0/5, all 26 others keep them 5/5.

## Review

None yet.

## Issues

None — no open issue; found by property testing. `snowflake` and `databricks` also lose slices, but their brackets mean element access, so `x[-4:-1]` becomes `x[-4 - 1]`. That needs real transpilation to `ARRAY_SLICE`, not a warning, so it was deliberately left out and offered as a separate issue.

## Outcome

Open with green CI, awaiting review. Third PR to this repo after [#8838](https://github.com/tobymao/sqlglot/pull/8838) and [#8844](https://github.com/tobymao/sqlglot/pull/8844), both merged.
