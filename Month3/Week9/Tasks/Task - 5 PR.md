
**Repo:** <https://github.com/ossf/scorecard>
**PR:** <https://github.com/ossf/scorecard/pull/5158>

---

## 1. Why this project

I ran OpenSSF Scorecard against my own `GoSystems` repo during the supply chain security track. Auditing a tool I already use.

---

## 2. Community Profile

```zsh
gh api repos/ossf/scorecard/community/profile \
  | jq '{health: .health_percentage, files: (.files | map_values(. != null))}'
```

```json
{
  "health": 87,
  "files": {
    "code_of_conduct": true,
    "code_of_conduct_file": true,
    "contributing": true,
    "issue_template": false,
    "pull_request_template": true,
    "license": true,
    "readme": true
  }
}
```

**87%.** The only reported gap is `issue_template`. Note that
`.github/ISSUE_TEMPLATE/` does exist in the repo — the API reports `false`
because it holds YAML issue *forms*, which the Community Profile endpoint does not count. The score understates the project here.

Governance files beyond the checklist: `CHARTER.md`, `MAINTAINERS.md`, `CONTRIBUTOR_LADDER.md`, `.github/CODEOWNERS`, `.github/security-insights.yml`,
`governance/meetings/` 

---

## 3. LICENSE

```zsh
gh api repos/ossf/scorecard/license --jq '.license.spdx_id'
curl -s https://raw.githubusercontent.com/ossf/scorecard/main/LICENSE | sed -n '189p'
#    Copyright 2020 OpenSSF Scorecard Authors
```

**Apache-2.0**

---

## 4. CONTRIBUTING.md

```zsh
gh api repos/ossf/scorecard/contents/CONTRIBUTING.md --jq '.content' | base64 -d | less
```

Hard requirements:

1. **DCO sign-off on every commit** (Linux Foundation). `git commit -s`.
2. **PR title prefixed with an emoji alias**: `:warning:` breaking,
   `:sparkles:` feature, `:bug:` fix, `:book:` docs, `:seedling:` infra,
   `:ghost:` no release note.
3. **No rebasing after review starts** — maintainers prefer merge commits.
4. **`docs/checks.md` is generated** from `docs/checks/internal/checks.yaml`;
   editing it directly is explicitly called out as wrong.

Local verification: `make all`, `make unit-test`, `make check-linter`,
`make fix-linter`, `make e2e-pat`.

Gap noticed: setup requires `go`, `protoc`, `make` before the contributing
steps, with no documentation-only path called out.

---

## 5. Finding the contribution

**Spelling scan:**

```zsh
pip install codespell --break-system-packages
codespell --skip="*.go,*.json,*.svg,*.sum,*.mod,vendor,third_party,testdata" .
```

Most hits were in `CHANGELOG.md`, `governance/meetings/*.md` and
`cron/internal/data/*.csv` — historical records and datasets, excluded.

**Broken link scan:** Python walk over every `.md`, extracting `[text](target)`,
skipping `http`/`mailto`/anchors, testing whether the path resolves. 8 candidates.

**Verification against live rendering:**

```zsh
curl -sL "https://github.com/ossf/scorecard/blob/main/MAINTAINERS.md" \
  | grep -oE 'href="[^"]*CHARTER[^"]*"'
# href="/ossf/scorecard/blob/main/CHARTER.md"
```

GitHub rewrites root-relative links (`/CHARTER.md`) to the repo root — 5 of the
8 were false positives, dropped. Only scheme-less external links are broken:

```zsh
curl -sL "https://github.com/ossf/scorecard/blob/main/docs/design/scalable_scorecard.md" \
  | grep -oE 'href="[^"]*oosf[^"]*"'
# href="/ossf/scorecard/blob/main/docs/design/github.com/oosf/scorecard"
```

### Findings

| # | File:line | Problem | Renders as |
| --- | --- | --- | --- |
| 1 | `docs/design/scalable_scorecard.md:11` | `[Scorecard](github.com/oosf/scorecard)` — org misspelled `oosf`, no scheme | `.../docs/design/github.com/oosf/scorecard` → 404 |
| 2 | `cron/k8s/README.md:6` | `[yamllint](yamllint.readthedocs.io)` — no scheme | `.../cron/k8s/yamllint.readthedocs.io` → 404 |
| 3 | `SECURITY.md:37` | "the alloted response window" | typo for "allotted" |

Left alone deliberately: a double space in `cron/k8s/README.md:10`, and the
`CHANGELOG.md` / meeting-minute typos.

---

## 6. The contribution

```zsh
gh repo fork ossf/scorecard --clone
cd scorecard
git checkout -b docs/fix-broken-links-and-typo

sed -i 's|\[Scorecard\](github.com/oosf/scorecard)|[Scorecard](https://github.com/ossf/scorecard)|' docs/design/scalable_scorecard.md
sed -i 's|\[yamllint\](yamllint.readthedocs.io)|[yamllint](https://yamllint.readthedocs.io)|' cron/k8s/README.md
sed -i 's/the alloted response window/the allotted response window/' SECURITY.md

git commit -s -a -m "docs: fix broken markdown links and a typo
...
"
# [docs/fix-broken-links-and-typo 956dc4c5] docs: fix broken markdown links and a typo
#  3 files changed, 3 insertions(+), 3 deletions(-)

git push -u origin docs/fix-broken-links-and-typo

gh pr create --repo ossf/scorecard \
  --title ":book: docs: fix broken markdown links and a typo" \
  --body-file /tmp/pr-body.md
# https://github.com/ossf/scorecard/pull/5158
```

- **Commit:** `956dc4c5`, DCO signed off with `-s`
- **PR #5158**, title prefixed `:book:` per the CONTRIBUTING emoji table
- Template filled: `Fixes: NONE`, `release-note: NONE`, tests checkbox N/A
- **Status:** open

