# Contribution 4 — SQLGlot

**Repo:** [tobymao/sqlglot](https://github.com/tobymao/sqlglot) · **PR:** [#8924](https://github.com/tobymao/sqlglot/pull/8924)
**Opened:** 6 Nov 2026 · **Closed:** 9 Nov 2026 by `georgesittas`, unmerged — *"Fixed in main"*; the fix shipped as his own commit [`cfda7864`](https://github.com/tobymao/sqlglot/commit/cfda7864e)

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

No review. Closed with *"Fixed in main"* and reimplemented by the maintainer the same day, in 3 lines instead of my 35:

```python
def slice_sql(self, expression: exp.Slice) -> str:
    if "[" in self.dialect.tokenizer_class._IDENTIFIERS:
        self.unsupported("Slices are not supported")
```

He read the condition straight off the tokenizer's identifier set, so no flag and no dialect files — where I had added `SUPPORTS_ARRAY_SLICING` to the base generator and set it `False` in five dialect generators. His version keeps my diagnostic string, my dialect list, and my test almost verbatim (same six dialects in the same order, same `assertLogs` + `assertRaises` shape), tightened with an assertion on the message text. The commit carries no reference to the PR and no co-author trailer.

**The lesson is mine to take.** My PR body argued the dialect list should come from a structural property rather than a judgement — "the correlation with bracket-quoting is exact, which is why the flag is set from that property" — and then I encoded that property as a hand-maintained table across six files instead of encoding the property itself. The maintainer encoded the property. A five-file flag derived from a one-line invariant is exactly the reviewable-surface objection `gregfurman` had already raised on bento #1116, in different clothing.

## Issues

None — no open issue; found by property testing. `snowflake` and `databricks` also lose slices, but their brackets mean element access, so `x[-4:-1]` becomes `x[-4 - 1]`. That needs real transpilation to `ARRAY_SLICE`, not a warning, so it was deliberately left out and offered as a separate issue.

## Outcome

Closed unmerged, but the fix shipped: `main` now warns on slices in bracket-quoting dialects, with my message and my test, under the maintainer's commit. The bug is fixed and the contribution is uncredited — the same pattern as boltons [#414](https://github.com/mahmoud/boltons/pull/414) and [#426](https://github.com/mahmoud/boltons/pull/426).

Third PR to this repo after [#8838](https://github.com/tobymao/sqlglot/pull/8838) and [#8844](https://github.com/tobymao/sqlglot/pull/8844), both merged.
