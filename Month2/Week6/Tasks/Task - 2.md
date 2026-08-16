**Project:** [B-textEditor](https://github.com/LD-RW/text-editor) — a modular, Kilo-inspired terminal text editor in C

---

## Task

Set up a professional documentation site with MkDocs Material for a personal repository, structure the content in Diátaxis style (exactly one Tutorial, one How-To guide, and Reference documentation), deploy it via GitHub Pages, and link it in the README.

---

## What I Built

A live docs site at **[ld-rw.github.io/text-editor](https://ld-rw.github.io/text-editor/)** — MkDocs Material theme, four pages (Home, Tutorial, How-To, Reference), auto-deployed by GitHub Actions on every push that touches `docs/**` or `mkdocs.yml`. The docs are grounded in the actual source: I read `src/input.c`, `src/find.c`, and `include/common.h` first, and found the README omitted real features (`Ctrl+S` save, `Ctrl+F` incremental search, the 3-press `Ctrl+Q` quit guard from `B_QUIT_TIMES`). The docs site documents what the code actually does.

---

## Steps

### 1. Local Environment

Used a venv instead of pacman/AUR — `mkdocs-material` isn't in the official Arch repos, and mixing AUR Python packages with pip causes conflicts:

```zsh
python -m venv .venv
source .venv/bin/activate
pip install mkdocs-material
echo '.venv/' >> .gitignore
```

### 2. Single-File Config

Wrote `mkdocs.yml` — site metadata, Material theme with light/dark palette toggle, `repo_url` + `edit_uri` (every page gets an edit pencil linking to GitHub), and the Diátaxis nav:

```yaml
nav:
  - Home: index.md
  - Tutorial: tutorial.md
  - How-to guides:
      - Add a keybinding: how-to-add-a-keybinding.md
  - Reference: reference.md
```

### 3. Content by Diátaxis Role

- **Tutorial** (learning-oriented): "Your first editing session" — build with `make`, open a file, edit, save with `Ctrl+S`, incremental search with `Ctrl+F`, quit through the 3-press guard. A guaranteed-success path for a newcomer.
- **How-To** (task-oriented): "Add a keybinding" — bind `Ctrl+G` to jump-to-top by touching three real files: prototype in `include/prototypes.h`, handler in `src/editor.c`, case in `editorProcessKeypress()` in `src/input.c`. Includes a warning admonition about reserved combos (`Ctrl+C`/`Ctrl+Z`/`Ctrl+D`).
- **Reference** (information-oriented): tables for every keybinding, the constants in `common.h` (`TAB_STOP`, `B_QUIT_TIMES`, version), all nine source modules and their responsibilities, and build targets/flags.

All content follows Google style: active voice, present tense, second person, sentence-case headings, no "easy/simply/just".

### 4. Local Verification

```zsh
mkdocs serve
```

Hot-reloading preview at `http://127.0.0.1:8000/text-editor/` — checked nav, dark-mode toggle, code-copy buttons, edit pencils, and key badges.

### 5. Deployment Workflow

`.github/workflows/docs.yml`: on push to `main` (path-filtered to `docs/**`, `mkdocs.yml`, and the workflow itself), checkout → setup Python 3.12 → cached `pip install mkdocs-material` → `mkdocs gh-deploy --force`. The job needs `permissions: contents: write` because `gh-deploy` force-pushes the built HTML to a `gh-pages` branch.

### 6. Pages + README

Order matters: push first, let the workflow's first run **create** the `gh-pages` branch, then Settings → Pages → Source: Deploy from a branch → `gh-pages` / root. Added a Documentation section to the README linking the live site.

---

## Problems I Ran Into (and Fixed)

|Problem|Root Cause|Fix|
|---|---|---|
|`Config value 'site_name': Required configuration not provided`|The pasted `mkdocs.yml` was flattened to a single line — the clipboard/editor stripped newlines, so YAML had no key structure|Recreated the file with a paste-proof heredoc (`cat > mkdocs.yml << 'EOF' ... EOF`); verified with `wc -l`|
|Raw `++ctrl+s++` pluses rendered as literal text in pages|The `pymdownx.keys` extension wasn't enabled in `markdown_extensions`, so its syntax passed through untouched|Added `- pymdownx.keys` to `mkdocs.yml`; no install needed since `pymdown-extensions` ships with `mkdocs-material`|
|`mkdocs-material` not installable via pacman|The package lives on PyPI/AUR, not in Arch's official repos|Project-local venv + pip; added `.venv/` to `.gitignore`|
|`gh-pages` branch missing from the Pages source dropdown|The branch only exists after the deploy workflow's first successful run|Waited for the Actions run to go green, then refreshed Settings → Pages|

---

## Key Concepts Learned

- **Diátaxis** — four documentation modes with distinct jobs: tutorials teach through a guaranteed-success path, how-tos solve one task for a competent user, reference states facts in lookup-friendly tables; mixing modes in one page weakens all of them
- **Docs must be sourced from code, not the README** — the README lagged the implementation; `input.c` was the ground truth for keybindings
- **`mkdocs gh-deploy`** — builds and force-pushes static HTML to `gh-pages`; the CI job needs `contents: write`, and Pages then serves that branch
- **`paths` filter on `push`** — doc-only workflow doesn't burn CI minutes on `src/` commits
- **`edit_uri`** — one config line gives every docs page an edit link back to the repo, closing the docs-as-code loop
- **`pymdownx.keys`** — `++ctrl+s++` renders as keyboard badges; extension syntax without the extension enabled is just literal text
- **Heredoc with quoted delimiter (`<< 'EOF'`)** — paste-proof file creation in zsh; the quotes prevent variable expansion inside the block
- **YAML fails structurally, Markdown fails silently** — a flattened YAML file errors immediately; a flattened Markdown file renders as one paragraph, so check both after a suspicious paste (`wc -l`)