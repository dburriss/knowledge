---
description: "Git worktrees let you have multiple branches checked out at once, each in its own directory, without extra clones. This guide covers setup, usage, and best practices for worktrees."
tags: [git]
---
# Git Worktrees

A _worktree_ lets one Git repository checkout multiple branches **simultaneously**, each in its own directory, without extra clones. It avoids the overhead of “stash → switch → unstash” and supports parallel feature work, quick hotfixes, and clean release builds.

Bare repos matter because **worktrees only work cleanly when the main repo is bare** or at least treated like one. Without that, the root directory becomes your “primary” checkout, which defeats the purpose of having independent worktrees.

## TLDR
Use a **bare repo** (`repo.git`) as the control repo.  
Create all actual working directories as **worktrees**.  
Run all Git commands **inside the bare repo directory**.

---

## Initial Setup

### Clone as bare

```sh
git clone --bare git@github.com:ORG/REPO.git repo.git
cd repo.git
````

---

## Creating Worktrees

### Checkout existing branch

Working dir _./repo.git_
```sh
git worktree add ../feature-123 feature/123
```

### Create + checkout new branch

```sh
git worktree add -b feature/xyz ../feature-xyz
```

### Checkout specific commit (detached)

```sh
git worktree add ../inspect abc123
```

### Temporary experiment

```sh
git worktree add -b spike/test ../spike
```

---

## Managing Worktrees

### List all

```sh
git worktree list
```

### Remove worktree

```sh
git worktree remove ../feature-123
```

### Force remove (dirty tree)

```sh
git worktree remove -f ../feature-xyz
```

### Clean up stale entries

```sh
git worktree prune
```

---

## Common Workflows

### Work on multiple features

```sh
git worktree add ../feat-a feature/a
git worktree add ../feat-b feature/b
```

### Hotfix without touching your feature branch

```sh
git worktree add ../hotfix main
```

---

## Notes / Pitfalls

- Do **not** delete worktree folders manually.
- A branch in a worktree is **locked** until you remove that worktree.
- Removing a worktree does **not** delete its branch.
- You cannot write code directly inside the bare repo; it has no working directory.

---

## Typical Directory Layout

```
project.git/                # bare repo (you run all git commands here)
project.git.main/           # worktree
project.git.feature-xyz/    # worktree
```
