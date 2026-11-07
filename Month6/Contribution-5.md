# Contribution 5 — Expat (libexpat)

**Repo:** [libexpat/libexpat](https://github.com/libexpat/libexpat) · **PR:** [#1493](https://github.com/libexpat/libexpat/pull/1493)
**Opened:** 7 Nov 2026 · **Status:** open, mergeable, CI awaiting maintainer approval (first PR to this repo) — 1 commit, 4 files, +20 / −6

---

## The contribution

Expat's allocation-failure tests drive the parser through its out-of-memory error paths, but they only assert that parsing failed *gracefully* — never that those paths released what they had allocated. `test_alloc_parse_xdecl`, for instance, loops the failure index and checks only that the parse neither succeeded immediately nor needed the maximum number of allocations.

Both halves of a leak detector were already in the suite and simply not connected:

- `duff_allocator` / `duff_reallocator` (`tests/common.c`) inject the failures, allocating with plain `malloc`/`realloc`.
- `tracking_malloc` / `tracking_realloc` / `tracking_free` and `tracking_report` (`tests/memcheck.c`) record what is still live.

`tracking_report` had exactly one caller in the entire suite, and `alloc_teardown` was just `basic_teardown`.

The change routes the duff allocators through the tracking allocator, frees through it in the four memory suites that pair with them, and makes `alloc_teardown` / `nsalloc_teardown` fail when anything is left over. Every existing allocation-failure test gains the check; no test logic or expectation changes.

## Goal

Make OOM-path leaks visible. A leak on an out-of-memory path is currently invisible to the suite, which is exactly the shape of their own [#1346](https://github.com/libexpat/libexpat/pull/1346) ("Fix OOM-related leak on a failed overflow check in `addBinding`") — nothing in the tests would have failed because of it.

Validated in both directions, since an assertion that never fires is worthless: the suite passes unmodified, and removing the `FREE(parser, b)` from the `b->uri` failure branch of `addBinding` — reintroducing a #1346-shaped leak — makes it fail with `Memory leak on an allocation-failure code path` instead of passing.

## Issues

Not fixing an open issue; related to #1346 as the bug class it would have caught. Found while hunting an actual leak: an external harness built on `XML_Memory_Handling_Suite` swept 66,000 combinations of expat's OOM paths (6 documents × 5 modes × 1200 failure indices) and found none, which is what pointed at the test suite's checking rather than the library.

## Outcome

Open, awaiting maintainer approval for CI. Test infrastructure rather than a leak fix — the hunt across libexpat, libarchive and yyjson (~70,000 injected-failure combinations) found exactly one real leak, in libarchive, and it was already covered by two open PRs there.
