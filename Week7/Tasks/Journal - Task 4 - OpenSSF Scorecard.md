
**Target project:** [mahmoud/boltons](https://github.com/mahmoud/boltons) (~6.5k ⭐, Python utility library)
**PR:** [#426 — chore(ci): pin GitHub Actions to commit SHAs](https://github.com/mahmoud/boltons/pull/426)

> [!abstract] Goal
> Run OpenSSF Scorecard on an open-source project (not my own), identify the lowest-scoring check, and try to open a PR fixing it.

## Why boltons

Not a random target: I'm already a contributor there (PR #414, a 17-test suite for `pathutils.py`). Drive-by security-config PRs from strangers often read as Scorecard badge-farming spam; from a known contributor they read as care. Before committing to the target, I scouted the repo directly — 10 unpinned `uses:` refs across `tests.yaml` and `publish.yml`, no `dependabot.yml`, no `SECURITY.md`, no top-level `permissions:` blocks — predicting several zero-scoring checks that an outsider can actually fix via PR.

## What I did

### 1. Ran Scorecard

Installed via `go install github.com/ossf/scorecard/v5@latest`, created a read-only classic PAT, exported it as `GITHUB_AUTH_TOKEN`, then:

```zsh
scorecard --repo=github.com/mahmoud/boltons --show-details | tee boltons-scorecard.txt
scorecard --repo=github.com/mahmoud/boltons --format=json > boltons-scorecard.json
```

### 2. Read the results

**Aggregate: 5.4 / 10.** The interesting split isn't high-vs-low scores — it's *fixable-by-outsider* vs *maintainer-only*:

| Score | Check | Outsider-fixable via PR? |
|---|---|---|
| 0/10 | **Pinned-Dependencies** | ✅ yes — pure config change |
| 0/10 | Token-Permissions | ✅ yes (2 lines/workflow) |
| 0/10 | Dependency-Update-Tool | ✅ yes (dependabot.yml) |
| 0/10 | Security-Policy | ✅ yes (SECURITY.md — but disclosure policy is the maintainer's to own) |
| 0/10 | SAST, Fuzzing, CII-Best-Practices | ❌ need maintainer buy-in / infrastructure |
| 6/10 | Code-Review | ❌ process, not config |
| 9–10/10 | CI-Tests, License, Maintained, Packaging, Contributors, Binary-Artifacts, Dangerous-Workflow, Vulnerabilities | — healthy |
| ? | Branch-Protection | errored (see problems); needs admin anyway |
| ? | Signed-Releases | no GitHub releases published |

**Chosen check: Pinned-Dependencies (0/10).** Among the tied zeros it has the highest real-world impact (the `tj-actions/changed-files` attack class — mutable tags re-pointed to malicious commits), requires zero behavioral change from the project, and I had just implemented the identical fix on my own repo — learned at home, contributed upstream.

### 3. Built the fix

Forked (fork already existed from #414), branched `chore/pin-actions-to-sha`, and resolved every tag to its real commit SHA with `git ls-remote` — checking for peeled `^{}` refs, since annotated tags list a tag-object SHA that must *not* be used for pinning. All ten pins stayed within the majors the maintainer already chose (v4/v5), only the ref made immutable, each with a `# vX.Y.Z` comment that Dependabot/Renovate machine-read to keep pins updated:

- `actions/checkout` → `34e11487…` (v4.3.1) — both workflows
- `astral-sh/setup-uv` → `d4b2f3b6…` (v5.4.2) — both workflows
- `actions/setup-python` → `a26af69b…` (v5.6.0) — both workflows
- `actions/upload-artifact` → `ea165f8d…` (v4.6.2)
- `actions/download-artifact` → `d3f86a10…` (v4.3.0)
- `pypa/gh-action-pypi-publish` → `cef22109…` (release/v1 head, pin date in comment)

Verified: `grep -rn "uses:" .github/workflows/ | grep -v "@[0-9a-f]\{40\}"` → `ALL PINNED ✔`, and `git diff` touched only `uses:` lines.

### 4. Measured the delta locally before pushing

```zsh
scorecard --local=. --checks=Pinned-Dependencies
```

**Result: 0 → 7 / 10.** All 10 of 10 action refs pinned; the remaining deduction is one `pipCommand` (see below).

### 5. Opened the PR

[PR #426](https://github.com/mahmoud/boltons/pull/426): single-concern diff (pins only), full threat-model explanation, and — importantly — the two judgment calls surfaced explicitly instead of smuggled through, with an offer to follow up separately with a `dependabot.yml`.

## Judgment calls (the part that isn't mechanical)

> [!tip] 1. Pinning a deliberately-moving branch
> `pypa/gh-action-pypi-publish@release/v1` is a branch pypa auto-updates so publishers get fixes. Pinning it is *more* secure — it executes at the most sensitive moment of the whole pipeline (PyPI publish) — but it transfers update responsibility to the maintainer. That's a semantics change, not a chore, so the PR flags it and offers to drop that one line. Never smuggle a trade-off past a maintainer inside a "chore" commit.

> [!tip] 2. The residual 7/10 is correct — don't chase 10
> The last Scorecard warning is `pipCommand not pinned` at `publish.yml:73`: `pip install boltons==$TAG` — the **post-publish smoke test** that installs the just-published boltons back from PyPI to verify propagation. Hash-pinning it is conceptually wrong: the artifact's identity *is the thing under test*. Contorting a smoke test to satisfy a scanner would be Goodhart's law — optimizing the metric instead of the security. Documented as an accepted residual in the PR instead.

## Problems I faced

> [!warning] Branch-Protection check errored: token scopes
> Scorecard's Branch-Protection check failed with `The 'rulesets' field requires one of the following scopes: ['public_repo'], but your token has only been granted the: [''] scopes` — and the whole run exited non-zero. I had created the PAT with *no* scopes (sufficient for most checks), but modern Scorecard queries repository rulesets via GraphQL, which needs `public_repo` even for public data. The error message literally names the missing scope and the settings URL — a textbook "read the error literally" fix. (Branch-Protection wasn't my fix target regardless: it requires repo admin.)

## Final result

- ✅ Scorecard executed against `mahmoud/boltons`: aggregate **5.4/10**, full per-check table saved (`boltons-scorecard.txt` / `.json`)
- ✅ Lowest-scoring *fixable* check identified from real output: **Pinned-Dependencies 0/10**
- ✅ [PR #426](https://github.com/mahmoud/boltons/pull/426) open upstream: 10/10 action refs pinned to verified commit SHAs, version comments preserved for Dependabot/Renovate, single-concern diff, trade-offs surfaced
- ✅ Local Scorecard confirms the delta: **Pinned-Dependencies 0 → 7** (residual = the intentionally-unpinned smoke test)

