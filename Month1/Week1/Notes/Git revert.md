#git #software_engineering #Progamming 

## Overview
The `git revert` command is a safe, non-destructive operation used to invert the changes of a specific target commit. Instead of altering or erasing project history, it creates a **brand-new commit** that applies the exact opposite changes of the specified commit.

> [!tip] Key Rule
> `git revert` **only** targets the changes in the specified commit hash. Any commits created after that target remain completely untouched and intact in your history.

---

## 🧠 How It Works (The "Anti-Matter" Commit)

If you have a history chain where you want to undo commit `C` (which sits behind subsequent work `D`, `E`, and `F`):

```text
A ─── B ─── C ─── D ─── E ─── F (HEAD)
````

Running `git revert [hash_of_C]` calculates the inverse of `C` and appends a new commit `C'` to the tip of your branch:

Plaintext

```
A ─── B ─── C ─── D ─── E ─── F ─── C' (HEAD)
```

- **Commits D, E, and F** are unaffected.
    
- **Commit C** remains visible in the log for historical accuracy.
    
- **Commit C'** neutralizes whatever code lines `C` originally introduced.
    

---

## 🛠️ How to Use It

### 1. Basic Revert (Auto-Commit)

Find the hash of the commit you want to undo using `git log --oneline`, then run:

Bash

```
git revert 976e130c
```

Git will automatically compute the inverse changes, open your default terminal text editor to let you edit the default revert message, and commit it immediately.

### 2. Revert Without Committing (`--no-commit`)

If you want to undo a commit but review the changes in your staging area (or make additional tweaks) before finalizing the commit, use the `-n` or `--no-commit` flag:

Bash

```
git revert -n 976e130c
```

---

## ⚠️ Handling Revert Conflicts

Because you are removing older changes while keeping newer modifications, Git might encounter a conflict if subsequent commits (`D`, `E`, or `F`) altered the exact same lines of code introduced by `C`.

When a conflict occurs, Git will pause the revert process and output a warning.

### Resolution Workflow:

1. Open your code editor and look for standard Git conflict markers:
    
    Plaintext
    
    ```
    <<<<<<< HEAD
    Current state of the code (from subsequent commits)
    =======
    State of the code trying to apply the inverse of the target commit
    >>>>>>> parent of 976e130c... Commit Message
    
    ```
    

```
2. Manually edit the files to resolve the conflicts.
3. Stage the resolved files:
   ```bash
   git add <file_path>
   
```

4. Continue the revert process:
    
    Bash
    
    ```
    git revert --continue
    
    ```
    

```
   *(If you get overwhelmed and want to abort the revert entirely, run `git revert --abort`)*

---

## 🥊 Git Revert vs. Git Reset

| Feature | `git revert` | `git reset` |
| :--- | :--- | :--- |
| **History Impact** | Preserves history (adds a new commit). | Rewrites history (erases or moves pointers backward). |
| **Collaboration** | Safe for shared, public remote branches. | Dangerous for shared branches; forces a `git push --force`. |
| **Scope** | Targets a single specific commit in isolation. | Affects everything from the target commit back to `HEAD`. |

---
## 🔗 Related Notes
- [[Git Stash]]
- [[Git Internals - Trees and Blobs]]
```