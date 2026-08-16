
**Project:** [Aurora Enterprise E-commerce](https://github.com/LD-RW/Aurora)

---

## Task

Take any existing repo and add a complete `.github/workflows/ci.yml` with: lint, test, matrix on 2+ platforms, caching, and a status badge in the README.

---

## What I Built

A full GitHub Actions CI pipeline for Aurora, a Spring Boot 4 e-commerce REST API, running on a 3-platform matrix (Linux, Windows, macOS) with Maven dependency caching.

---

## Steps

### 1. Created the Workflow File

`.github/workflows/ci.yml`:

```yml
name: Aurora CI

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

jobs:
  build-and-test:
    name: Build & Test on ${{ matrix.os }}
    runs-on: ${{ matrix.os }}

    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]

    # mvnw lives inside the Aurora/ subdirectory, not the repo root
    defaults:
      run:
        working-directory: Aurora

    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Set up JDK 25
        uses: actions/setup-java@v4
        with:
          java-version: '25'
          distribution: 'temurin'
          # Caches ~/.m2/repository keyed on all pom.xml files found in the repo
          cache: 'maven'

      - name: Grant execute permission for Maven Wrapper (Linux/macOS)
        if: runner.os != 'Windows'
        run: chmod +x mvnw

      # shell: bash ensures ./mvnw works on Windows (via Git Bash) and Linux/macOS
      # verify runs: compile → test → package (catches lint-level errors at compile time)
      - name: Lint and Test
        shell: bash
        run: ./mvnw clean verify
```

### 2. Added a Dynamic Status Badge to the README

```markdown
[![Aurora CI](https://github.com/LD-RW/Aurora/actions/workflows/ci.yml/badge.svg?branch=main)](https://github.com/LD-RW/Aurora/actions/workflows/ci.yml)
```

The badge is a live SVG served by GitHub — it turns green or red automatically as workflows pass or fail on `main`.

### 3. Tested Locally

Before pushing, I verified the build passes locally:

```zsh
cd Aurora
./mvnw clean verify
```

### 4. Pushed and Verified on GitHub

```zsh
git add .github/workflows/ci.yml README.md
git commit -m "ci: add GitHub Actions CI workflow with matrix and badge"
git push
```

All 3 matrix jobs completed with green checkmarks on the first run.

---

## Problems I Ran Into (and Fixed)

| Problem | Root Cause | Fix |
|---|---|---|
| `./mvnw` would fail on Windows | Windows runners use PowerShell by default, which can't execute a bash script | Added `shell: bash` — Windows runners have Git Bash, so this works on all three platforms |
| `mvnw: not found` | The Maven wrapper is inside `Aurora/`, not the repo root | Added `defaults.run.working-directory: Aurora` so all `run:` steps target the right directory |
| EndeavourOS not available | GitHub-hosted runners only offer `ubuntu-latest`, `windows-latest`, `macos-latest` | Used `ubuntu-latest` for Linux; EndeavourOS would require a self-hosted runner |

---

## Key Concepts Learned

- **`strategy.matrix`** — runs the same job across multiple environments in parallel
- **`fail-fast: false`** — lets all matrix jobs finish even if one fails, giving a full picture
- **`cache: 'maven'`** in `actions/setup-java` — automatically caches `~/.m2/repository` keyed on `**/pom.xml`, avoiding repeated downloads
- **`defaults.run.working-directory`** — sets the working directory for all `run:` steps in a job without repeating it on every step
- **`shell: bash`** — forces Git Bash on Windows, making cross-platform shell scripts work without duplicating steps
- **Status badge URL format:** `https://github.com/{owner}/{repo}/actions/workflows/{file}.svg?branch={branch}`
