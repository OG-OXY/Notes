# Jujutsu (`jj`) Multi-Device Sync & Workflow Guide

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
