#git #software_engineering #Progamming 
# Git Stash

The `git stash` command acts as a **temporary pause button** for your workspace. It takes your uncommitted changes (both staged and unstaged), saves them to an internal Git stack, and reverts your working directory to a pristine state matching the `HEAD` commit.

> [!info] The Core Use Case
> Use `git stash` when you are in the middle of a task with messy, half-written code and you need to switch branches immediately (e.g., to fix an urgent bug or pull upstream updates) without creating a messy, incomplete commit.

---

## 🛠️ Essential Workflow

### 1. Save Your Work
To save your current modifications and clean your working directory:
```bash
git stash
````

To give your stash a recognizable name (strongly recommended):

Bash

```
git stash save "WIP: working on aurora layout"
# OR (Modern Git)
git stash push -m "WIP: fixed terminal parsing logic"
```

### 2. View Saved Stashes

To see a history of everything you have stashed on your local stack:

Bash

```
git stash list
```

_Output format example:_ `stash@{0}: On main: WIP: fixed terminal parsing logic` `stash@{1}: On dev: WIP: working on aurora layout`

### 3. Restore Your Work

When you are ready to bring your changes back, switch to your target branch and run:

Bash

```
# Apply the latest stash AND remove it from the stack
git stash pop

# Apply the latest stash BUT keep it on the stack
git stash apply
```

To apply a specific stash from your list instead of the most recent one:

Bash

```
git stash pop stash@{1}
```

---

## ⚡ Advanced & Useful Flags

### Handling Untracked Files

By default, `git stash` ignores brand-new files that haven't been staged or tracked yet. To include them, use the `--include-untracked` or `-u` flag:

Bash

```
git stash -u
```

### Keeping the Staged Index

If you already ran `git add` on some files and want to keep those staged while stashing the rest of your unstaged modifications:

Bash

```
git stash --keep-index
```

### Inspecting a Stash

To see what changes are inside a stash before applying it:

Bash

```
# See a summary of changed files
git stash show

# See the full diff of the changes
git stash show -p
```

---

## 🧹 Cleanup

If you applied a stash using `apply` and want to delete it manually, or if you just want to clear out your old work:

Bash

```
# Delete a specific stash
git stash drop stash@{0}

# Delete ALL stashes on the stack
git stash clear
```

---