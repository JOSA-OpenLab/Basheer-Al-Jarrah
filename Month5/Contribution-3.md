# Contribution 3 — marshmallow

**Repo:** [marshmallow-code/marshmallow](https://github.com/marshmallow-code/marshmallow) · **PR:** [#3059](https://github.com/marshmallow-code/marshmallow/pull/3059)
**Opened:** 28 Sep 2026 · **Status:** open, waiting on the first-time-contributor CI gate

---

## What I did

Made `Schema.load` 10–14% faster by not constructing a validator object for fields that have no validators. The whole diff is two lines.

## Why

marshmallow is what turns incoming JSON into validated Python objects. In a web service, `load` runs on every request body, and webargs, apispec and flask-marshmallow all sit on top of it, so anything on that path is inherited by a lot of code.

`Field._validate` runs once per field per object. It looked harmless:

```python
def _validate(self, value):
    self._validate_all(value)

@property
def _validate_all(self):
    return And(*self.validators)
```

`_validate_all` is a **property**, so that one line allocates a new `And` object and a tuple of validators, calls it, and the call allocates a list and a dict of its own. For a field with no validators, all of it exists to iterate over nothing and then get thrown away. A seven-field schema loading a thousand records does that seven thousand times, and `validate=` is opt-in, so most fields in most schemas never have one.

So: ask whether there's anything to do before doing it.

```python
if self.validators:
    self._validate_all(value)
```

## How

The safety argument is short. `And()` with zero validators builds an empty error list, loops zero times, raises nothing and returns. Skipping it cannot change a result.

The part I had to actually check was `Email` and `URL`. Both call `self.validators.insert(0, ...)` in their own `__init__`, *after* `super().__init__()` has run — so the list isn't fixed at construction time. That rules out caching anything at construction, but the guard reads the list at validation time, so it sees those inserts fine. I confirmed both still reject bad input end to end.

To be sure there was nothing else, I ran the new `_validate` against a verbatim copy of the old body over 2,178 combinations of field type, validator configuration and input value, comparing the exception type *and* the full error messages and kwargs. No differences. Full suite: 1,190 passed, with ruff and mypy clean.

**The interesting bit.** This has been there since 2021, when someone replaced an inline validation loop with `And(...)` — a readability refactor, and a good one. The allocation was an unintended side effect. It survived five years because `performance/benchmark.py` **only measures `dump`**, and `_validate` is reached only from `deserialize`. The project's own performance harness structurally cannot see this path. So I mirrored their harness onto `load` using their exact schemas, and reported `dump` alongside as a control to show it was genuinely untouched rather than lost in noise.

A useful check on my setup: my baseline `dump` came out at 8.96 µs against the 8.9 they published in PR #3024, so I knew I was measuring the same thing they were.

## Numbers

Minimum of three interleaved before/after runs, `--iterations=100 --repeat=5`.

| Python | load: before | load: after | | dump (control) |
| --- | ---: | ---: | --- | --- |
| 3.14 | 15.84 µs | 13.64 µs | −13.9% | 8.98 → 9.03 |
| 3.13 | 13.10 µs | 11.35 µs | −13.4% | 7.21 → 7.16 |
| 3.12 | 13.73 µs | 12.16 µs | −11.4% | 7.71 → 7.79 |
| 3.10 | 20.16 µs | 17.99 µs | −10.8% | 12.39 → 12.46 |

For scale, the last performance PR they merged was worth 3.4%.
