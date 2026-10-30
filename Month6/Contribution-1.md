# Contribution 1 — Bento

**Repo:** [warpstreamlabs/bento](https://github.com/warpstreamlabs/bento) · **PR:** [#1116](https://github.com/warpstreamlabs/bento/pull/1116)
**Opened:** 23 Oct 2026 · **Closed:** 30 Oct 2026 by `gregfurman`, unmerged

---

## The contribution

Bento compiles every interpolated config field (output topics, keys, S3 paths, cache keys) into a `field.Expression` that resolves once per message. `Expression.String` was implemented on top of `Expression.Bytes`, so a query returning a string was copied to `[]byte` and back on every resolution. The `Resolver` interface already declared `ResolveString` and both resolvers implemented it, but nothing called it.

The PR routed `String` through that unused string path and added a buffer pre-allocation hint (`sizeHint` + `buf.Grow`).

## Goal

Remove the per-message round trip on the hot path — `TryString` has 102 call sites against `TryBytes`'s 8. Allocations halved (4 → 2 on a metadata lookup), and a long object-storage path went from 659 ns to 295 ns.

## Review

Declined. `gregfurman` accepted the round-trip diagnosis but rejected the patch on surface area, and specifically rejected the pre-allocation: `sizeHint` is derived from static text length, so it stops helping exactly when a dynamic segment resolves to something large — the large-payload tail case, which I never benchmarked. He preferred reworking the interfaces instead, splitting resolvers via generics into `Resolver[string]` and `Resolver[[]byte]` with a single `Resolve` method.

No changes were made to the PR; it was closed as-is.

## Issues

None when opened. After the close I raised [#1132](https://github.com/warpstreamlabs/bento/issues/1132) to carry the profile data and the two constraints the generics design has to clear: `InterpolatedString` exposes both `TryString` and `TryBytes` on the same object, and resolvers are only built through `Func[field.Resolver]` parser combinators.

## Outcome

Closed unmerged. Nothing shipped. The round trip is still there, and the discussion continues in #1132.
