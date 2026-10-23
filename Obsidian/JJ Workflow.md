y# Jujutsu (`jj`) Multi-Device Sync & Workflow Guide

A complete reference guide for managing a notes repository across devices using Jujutsu, balancing linear publishing with multi-parent DAG merging.

---
## 1. Core Mental Models: Git vs. Jujutsu

* **Fetching (`jj git fetch` / `git fetch`):** Downloads remote branches and updates your local tracking view (`origin/master`) without modifying your working files or local branch.
* **Working Copy (`@`):** In Jujutsu, your working directory is *always* a live commit. You never need to "stage" or "commit" just to save work.

---
## 2. The Two Core Workflows
### Scenario A: Sequential / Finished Work (The Standard Routine)
Use this when you work sequentially (e.g., make changes on your phone, push them, come to your PC, and pull them down without overlapping simultaneous edits).
1. **Pull changes (PC):**
```bash
jj git fetch
jj rebase -d origin@master

```
2. **Push changes (When 100% finished):**
```bash
jbs  # jj bookmark set master -r @
jdc  # jj describe -m "Commit message"
jgp  # jj git push --all --allow-empty-description

```
### Scenario B: Divergent / "Split-Brain" Work (The DAG Merge)
Use this when your timelines diverge (e.g., you edited notes on your PC *and* your phone without syncing first) and you need to safely merge both states without rewriting history.
1. **Pull and Merge natively:**
```bash
jj git fetch
jmc  # jj new @ origin@master

```
* *What this does:* Spawns a new working-copy change with **two parents** (`@` and `origin@master`). If files overlap, Jujutsu drops standard conflict markers right into your working files for easy in-place resolution.
---
## 3. Iterative Chunking with `jj commit` (`jco`)
Instead of waiting until the very end to describe your work, you can lock in changes instantly on the fly:
```bash
jco "Finished section on NixOS modules"

```
* **What it does:** Snapshots your current working copy into a permanent immutable commit and instantly hands you a fresh, empty working copy (`@`) on top of it.
* **History Safety:** All intermediate commits stack safely underneath your master bookmark. Jujutsu also tracks everything via `jj op log`, ensuring nothing is ever permanently lost.
---
## 4. Complete Abbreviation Reference

| Abbreviation | Expanded Command                              | Purpose                                                                     |
| ------------ | --------------------------------------------- | --------------------------------------------------------------------------- |
| **`jbs`**    | `jj bookmark set master -r @`                 | Snaps the local `master` bookmark to your current working commit.           |
| **`jdc`**    | `jj describe -m ""`                           | Adds/updates the description message for the current commit.                |
| **`jgp`**    | `jj git push --all --allow-empty-description` | Pushes all bookmarks up to the remote repository.                           |
| **`jmc`**    | `jj new @ origin@master`                      | Creates a true multi-parent DAG merge commit between local and remote.      |
| **`jco`**    | `jj commit -m`                                | Locks current work into a permanent commit and spawns a fresh working copy. |
The reason those hashes aren't matching up is because you are comparing **two completely different layers of history in Jujutsu**:

1. **`jj log` / `jj status`:** Deals with **Commits / Revisions** inside your DAG tree (`mkrxykxw` / `d0b60b2a`).
2. **`jj op log`:** Deals with **Operations** in the entire repo metadata (`9e91c612530c` / `2bdbd58c6063`).

---

# JJ & Git Commit Hashes and Operation ID's
### Breakdown of What Each Hash Is

#### 1. The Right Number in `jj status`: `d0b60b2a` (Commit ID)

* **What it is:** The Git commit hash of the snapshot.
* **Where it lives:** Inside the commit tree DAG (`jj log`).
* **Why it changes:** If you amend or add files to `@`, `d0b60b2a` turns into a new Git hash because the content changed.

#### 2. The Hashes in `jj op log`: `9e91c612530c` & `2bdbd58c6063` (Operation IDs)

* **What they are:** Unique IDs for **the action you took on the repository itself** (e.g., "running `jj commit`", "snapshotting working copy", "rebasing").
* **Why they don't match `d0b60b2a`:** An Operation ID is **NOT a commit hash**. An Operation is a time-capsule of the *entire repository state* at the instant you ran a CLI command.

Look at your terminal output:

```text
9e91c612530c   1 hour ago    snapshot working copy
2bdbd58c6063   4 minutes ago commit 6159d1f5132a048299cae7b073f22c17469687fe

```

* `9e91c612530c` is the **Operation ID** of when `jj` automatically snapshotted your files 1 hour ago.
* `2bdbd58c6063` is the **Operation ID** generated 4 minutes ago when you ran `jj commit -m "..."`.
* `6159d1f...` printed at the far right is the **Full 40-character Commit ID** created *by* that operation!

---

### How to Use Each Hash with Commands

#### Scenario A: Manipulating Code / Commits (Use Change ID or Commit ID)

When you want to edit, rebase, diff, or look at a specific commit in your tree:

* **Command:** `jj edit`, `jj diff -r`, `jj rebase -r`
* **Which ID:** Use the **Left Change ID** (`mkrxykxw`) or **Right Commit ID** (`d0b60b2a`).

```bash
jj diff -r mkrxykxw       # Works on the Change ID (Left)
jj show d0b60b2a          # Works on the Commit ID (Right)

```

#### Scenario B: Time-Traveling / Undoing Actions (Use Operation ID)

When using `jj op log` or your fzf bind to restore the entire repo back to a previous state in time (like undoing a bad rebase or accidental deletion):

* **Command:** `jj op restore`, `jj op show`, `jj --at-op`
* **Which ID:** Use the **Operation ID** (`2bdbd58c6063`) from `jj op log`.

```bash
# Restore the entire repository state back to how it was before you ran the last command:
jj op restore 2bdbd58c6063

# Or inspect what the log looked like at that specific moment in time:
jj log --at-op 2bdbd58c6063

```

---

### Summary Checklist

| ID Type          | Example from your screenshot | Where it comes from   | What it's used for                                          |
| ---------------- | ---------------------------- | --------------------- | ----------------------------------------------------------- |
| **Change ID**    | `mkrxykxw` (Left)            | `jj log`, `jj status` | Daily code edits, rebasing, describing.                     |
| **Commit ID**    | `d0b60b2a` (Right)           | `jj log`, `jj status` | Exact Git commit object hash.                               |
| **Operation ID** | `2bdbd58c6063`               | `jj op log`           | Restoring repo history / undoing actions (`jj op restore`). |
