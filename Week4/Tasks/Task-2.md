## Self-Review: jemalloc PR #2883

**Repo:** https://github.com/jemalloc/jemalloc **PR:** `bin: enforce bin->lock ownership in bin_slab_reg_alloc() to prevent bitmap race` **PR link:** https://github.com/jemalloc/jemalloc/pull/2883

### Maintainer's feedback (lexprfuncall)

- No bug is actually fixed — both callers already hold the lock and already check it; this just adds a redundant assertion for debug builds.
- Breaks naming convention — jemalloc marks lock-required functions with a `_locked` suffix; this PR instead threads new parameters through the signature instead.
- The unrelated AppVeyor/CI fix should have been its own PR, not bundled in.

### My review (third person)

The contributor correctly traces the bitmap race down to its root cause — two threads racing on `bitmap_sfu()` → `bitmap_set()` — and the analysis is technically sound. But the PR overclaims: it's framed as preventing a race, when in fact both existing callers already enforce the lock contract, so the change is a debug-mode hardening, not a bug fix. The contributor also deviated from the codebase's established `_locked` naming convention without justifying the deviation, and folded an unrelated CI fix into the same diff, increasing review surface for no real benefit. The underlying engineering instinct was good; the packaging and framing weren't ready to ship.