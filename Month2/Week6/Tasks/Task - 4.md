**Repo:** [Aurora Enterprise E-commerce](https://github.com/LD-RW/Aurora)

---

## Task

Add Vale to one repo: configure it with the Microsoft Writing Style Guide, run it on the docs, fix what it finds, and integrate it into the existing GitHub Actions CI workflow (the same `ci.yml` built in the earlier CI task).

---

## What I Built

A prose-linting gate for Aurora's documentation (`README.md` + two ADRs under `Aurora/docs/adr/`). Vale with the Microsoft style found **13 errors, 11 warnings, and 24 suggestions** on the first run. After fixing the genuine errors and absorbing the false positives with a project vocabulary, the error count is zero, and a new `prose-lint` job in `ci.yml` fails the build on any future style error — the "style violations fail CI" loop from docs-as-code, now real.

---

## Steps

### 1. Configured at the Repo Root

`.vale.ini` sits at the repo **root** (not inside `Aurora/`) so it covers both `README.md` and `Aurora/docs/`:

```ini
StylesPath = .vale/styles
MinAlertLevel = suggestion

Packages = Microsoft
Vocab = Aurora

[*.md]
BasedOnStyles = Vale, Microsoft
```

`vale sync` downloads the Microsoft package into `.vale/styles/`. Git hygiene: the synced package is gitignored (re-downloadable, like `node_modules`); the vocabulary is committed.

### 2. Triaged the First Run

`vale README.md Aurora/docs/` → 13 errors in three categories, each demanding a different response:

- **8× `Microsoft.Foreign`** — `e.g.,` must become `for example` → genuine fix, edit the prose
- **1× `Microsoft.Contractions`** — `is not` must become `isn't` → genuine fix
- **4× `Vale.Spelling`** — `Lombok's`, `DTOs` (×2), `declaratively` → false positives; correct words Vale's dictionary doesn't know → fix the *tooling*, not the prose

### 3. Fixed Prose, Taught Vocabulary

```zsh
sed -i 's/e\.g\., /for example, /g; s/e\.g\.,/for example,/g' Aurora/docs/adr/*.md
```

Vocabulary at `.vale/styles/config/vocabularies/Aurora/accept.txt` (Aurora, Lombok, Lombok's, DTO, DTOs, declaratively). Verified:

```zsh
vale --minAlertLevel=error README.md Aurora/docs/
# ✔ 0 errors, 0 warnings and 0 suggestions in 3 files.
```

The remaining 11 warnings (first-person "I", passive voice, heading capitalization) stay by design — ADRs legitimately speak in first person, so the CI gate is errors-only.

### 4. Tested the Gate Before Pushing

Three levels, cheapest first:

1. **Exit codes** — CI pass/fail is just the last step's exit code. Clean run → `echo $?` = 0. **Negative test:** appended `This is bad, e.g., this phrase.` to the README → Vale flagged `README.md 6:14 Microsoft.Foreign`, exit 1 → `git checkout --` to restore. A gate that has never failed is untested.
2. **actionlint** — statically validates the workflow YAML (schema, indentation, shellcheck on `run:` blocks). Silence = pass.
3. **act** (skipped as overkill here) — full Docker rehearsal of the job; worth it only for workflows with secrets or complex step interdependencies.

### 5. Added the CI Job

Second job in the existing `.github/workflows/ci.yml`, parallel to `build-and-test`:

```yaml
  prose-lint:
    name: Prose Lint (Vale)
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4
      - name: Install Vale
        run: curl -sL https://github.com/errata-ai/vale/releases/download/v3.15.1/vale_3.15.1_Linux_64-bit.tar.gz | sudo tar xz -C /usr/local/bin vale
      - name: Sync Vale styles
        run: vale sync
      - name: Lint prose
        run: vale --minAlertLevel=error README.md Aurora/docs/
```

Design decisions: **no matrix** (prose is platform-independent; it runs as a parallel ~20-second job, not 3×), **no `working-directory: Aurora`** (unlike `mvnw`, the config deliberately lives at root to cover the README), **pinned v3.15.1** (release assets embed the version in the filename, so "latest" URLs don't exist anyway, and pinning keeps CI reproducible).


## Problems I Ran Into (and Fixed)

| Problem | Root Cause | Fix |
|---|---|---|
| Vale flagged `Lombok's`, `DTOs`, `declaratively` as spelling errors | Vale's dictionary doesn't know domain/project terms — these are false positives, not prose bugs | Project vocabulary in `.vale/styles/config/vocabularies/Aurora/accept.txt` + `Vocab = Aurora` in the ini; never reword correct prose to appease a linter |
| `Microsoft.FirstPerson` warned on every "I" in the ADRs | ADRs are legitimately first-person decision records; the Microsoft style targets product docs | Gated CI on `--minAlertLevel=error` so warnings stay advisory; per-glob rule tuning (`[Aurora/docs/adr/*.md]` → `Microsoft.FirstPerson = NO`) available if the noise bothers me locally |
| Vale not installable via pacman | Not in Arch official repos, only AUR | `yay -S vale-bin` (EndeavourOS ships yay); binary tarball as fallback |
| No stable "latest" download URL for CI | Vale release assets embed the version in the filename (`vale_3.15.1_Linux_64-bit.tar.gz`) | Pinned the exact version in the workflow — also makes CI reproducible |
| How to trust the gate before pushing | A green check on clean docs proves nothing about the failure path | Negative test: injected `e.g.,`, confirmed exit code 1 and the exact `file line:col rule` output, reverted with `git checkout --` |
| Repo path contains spaces (`Software Engineering/E-commerce`) | Personal directory layout | No effect on Vale, git, or CI (runners clone to their own path) — but noted as a hazard for any future unquoted `$PWD` in shell scripts |
