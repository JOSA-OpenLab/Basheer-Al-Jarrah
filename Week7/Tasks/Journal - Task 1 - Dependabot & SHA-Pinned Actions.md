
**Repo:** [LD-RW/GoSystems-High-Performance-HTTP-1.1-Server](https://github.com/LD-RW/GoSystems-High-Performance-HTTP-1.1-Server)
**PR:** [#1 — chore(security): Dependabot + SHA-pinned Actions](https://github.com/LD-RW/GoSystems-High-Performance-HTTP-1.1-Server/pull/1)

> [!abstract] Goal
> Configure Dependabot for both a code ecosystem (`gomod`) and the `github-actions` ecosystem, pin all GitHub Actions to full commit SHAs instead of mutable tags, and submit everything as a PR.

## What I did

1. **Picked the target repo.** GoSystems (my Go HTTP/1.1 server) — a real project with a `go.mod` at the root, so the code ecosystem maps cleanly to `gomod`.
2. **Created a branch** `chore/dependabot-and-sha-pinning` off `main`.
3. **Added `.github/dependabot.yml`** with two update blocks:
   - `gomod` at `/`, weekly, PR limit 10, commit prefix `chore(deps)`
   - `github-actions` at `/`, weekly, PR limit 10, commit prefix `chore(ci)`
   - `directory: /` for the `github-actions` ecosystem is the convention — it scans `.github/workflows/` automatically.
4. **Discovered the repo had no workflows at all** (`ls .github/workflows` → nothing). "Pin all your Actions" is vacuous with zero Actions, so I created a real CI workflow (`.github/workflows/ci.yml`) that builds, vets, and tests the Go module — *born pinned*:
   - `actions/checkout@df4cb1c069e1874edd31b4311f1884172cec0e10 # v6.0.3`
   - `actions/setup-go@924ae3a1cded613372ab5595356fb5720e22ba16 # v6.5.0`
   - Also scoped the workflow token with `permissions: contents: read` (minimal `GITHUB_TOKEN`, the Scorecard *Token-Permissions* check).
5. **Resolved SHAs correctly.** Tags were resolved with `git ls-remote`, using the *peeled* (`^{}`) commit SHA where the tag is annotated — pinning to a tag-object SHA instead of the commit SHA would break resolution. Kept the `# vX.Y.Z` trailing comments because Dependabot machine-reads them to bump SHA + comment together.
6. **Validated before pushing:**
   - `python3 -c "import yaml; yaml.safe_load(...)"` → `dependabot.yml OK`
   - `grep -rn "uses:" .github/workflows/ | grep -v "@[0-9a-f]\{40\}"` → empty → `ALL PINNED ✔`
7. **Committed with a conventional commit** (`chore(security): ...`) explaining the threat model, pushed, and opened PR #1 with `gh pr create`.
8. **Merged the PR** after CI passed, which activated Dependabot.

## Problems I faced

> [!warning] Dependabot showed "not configured yet" after opening the PR
> After pushing the branch and opening PR #1, the repo's **Insights → Dependency graph → Dependabot** tab still said *"Dependabot version updates aren't configured yet."*
>
> **Root cause:** Dependabot only reads `.github/dependabot.yml` from the **default branch**. The config existed only on my feature branch — an unmerged PR is invisible to Dependabot.
>
> **Fix:** merge PR #1 into `main`. After the merge, the Dependabot tab picked up both ecosystems.

## Final result

- ✅ PR #1 merged into `main`
- ✅ `.github/dependabot.yml` live with **gomod** and **github-actions** ecosystems (weekly)
- ✅ `.github/workflows/ci.yml` — CI (build / vet / test) with **100% of Actions pinned to commit SHAs** with version comments
- ✅ Workflow `GITHUB_TOKEN` scoped to `contents: read`
- ✅ Dependabot tab (Insights → Dependency graph → Dependabot) shows both ecosystems with "Last checked" timestamps

Final state on `main`:

```
.github/
├── dependabot.yml
└── workflows/
    └── ci.yml   # every uses: = owner/action@<40-hex-sha> # vX.Y.Z
```

> [!tip] Scorecard side-effects
> This single PR improves three OpenSSF Scorecard checks at once: **Pinned-Dependencies**, **Dependency-Update-Tool**, and **Token-Permissions** — useful groundwork for Task 4.
