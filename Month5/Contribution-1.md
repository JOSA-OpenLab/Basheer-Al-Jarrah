# Contribution 1 — Pygments

**Repo:** [pygments/pygments](https://github.com/pygments/pygments) · **PR:** [#3320](https://github.com/pygments/pygments/pull/3320)
**Opened:** 21 Sep 2026 · **Status:** open, mergeable, CI awaiting maintainer approval (first PR to this repo)

---

## The repo

Pygments is the syntax highlighter most of the Python world runs on. Sphinx uses it, so it renders the code in nearly every Python project's docs; Jupyter, GitHub, and countless static site generators use it too. It ships around 550 lexers, each one a set of regular expressions that turn source text into coloured tokens.

That reach is what makes its bugs matter. A lexer flaw doesn't just affect Pygments — it affects every site and tool that highlights untrusted code through it.

## What I did

I fixed an exponential-time regex in the Maple lexer that lets a 45-byte file hang the process for over a minute.

The string rule was `"(\\.|.|\s)*?"`. The `.` branch also matches a backslash, and `\s` overlaps `.`, so a run of backslashes can be split between the branches in exponentially many ways. On an unterminated string the engine tries every split before giving up. It is the same bug family as CVE-2021-27291.

I found it by scanning all 263 lexer modules and, instead of trusting a pattern match, feeding each suspicious regex adversarial input at growing lengths and timing it. That distinction mattered: `(\\.|[^"])*` explodes while `(\\.|[^"\\])*` is fine, and nothing but measurement separates them. 507 static hits reduced to 8 lexers that actually misbehaved through the real lexer API.

Seven of those eight are already covered by an open PR (#3228). Maple is not.

The fix makes the branches disjoint:

```python
(r'"(\\[\s\S]|[^"\\])*"', String),
```

Measured through the `pygmentize` CLI on my machine (i7-12700H, Python 3.14.7):

| file size | before | after |
| ---: | ---: | ---: |
| 37 bytes | 2.77 s | 0.03 s |
| 41 bytes | 18.63 s | 0.06 s |
| 45 bytes | > 60 s, aborted | 0.06 s |

## Why the PR is worth it

It closes a real denial of service. Any service that highlights user-supplied Maple snippets can be stalled indefinitely by a few dozen bytes, and the cost multiplies about 6.7× for every four extra characters.

It is the one instance the existing cleanup missed. #3228 fixes nine lexers and has been open since July; maple falls outside it and would otherwise stay vulnerable after that PR lands.

The change is one line, and I know exactly what it affects. I compared the old and new patterns across 157,656 inputs — exhaustive to length 7 over a hostile alphabet, plus 60,000 random. 1,671 differ, and every one is a case where the old pattern matched and the new one doesn't. Those are strings closable only by treating a trailing backslash as literal, like `"abc\"`. Maple reads `\"` as an escaped quote, so they are genuinely unterminated: the fix corrects a mis-lex as well as the blowup. Both Maple example files tokenise byte-identically.

It carries a regression test that actually fails without it — the new snippet test doesn't finish inside 60 seconds on the unpatched lexer. The full suite passes (5331 tests) and `regexlint`, the gate CI runs, is clean.
