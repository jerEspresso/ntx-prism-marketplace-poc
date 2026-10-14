---
description: List all /worktree commands and what they do
allowed-tools:
---

Print the worktree command reference below to the terminal **verbatim** as your
entire response. Do NOT run any tools, inspect the repo, or add commentary —
this command is purely informational.

---

## `/worktree` — bare + worktree repo commands

A **bare + worktree** repo (under `X:\repos\<name>`) keeps one shared
object store in `.bare` and checks out each branch into its own subfolder, so
multiple branches can be open at once for parallel sessions. Point each Claude
Code / VS Code window at a **worktree subfolder**, never the container root.

### Setup

- **`/worktree:clone <git-url> [folder-name]`**
  Clone a repo into the bare + worktree layout. Sets up the `.bare` store, the
  `.git` pointer, remote tracking refspec, and a worktree for the default
  branch (with upstream tracking).

- **`/worktree:convert [path]`**
  Convert an existing normal clone in-place into the bare + worktree layout.
  Non-destructive.

### Daily use

- **`/worktree:add [branch | work-item# | description] [folder-name]`**
  Add a worktree alongside the others. Accepts an explicit branch name, a work
  item number (title fetched from Azure DevOps to build the name), a plain
  description of the change, or nothing (inferred) — I name it per the
  convention in `X:\CLAUDE.md`. Sets up upstream tracking when a matching remote
  branch exists.

- **`/worktree:branch [new-branch | work-item# | description] [folder-name]`**
  Spin the current worktree's in-progress work onto a new branch in its own
  worktree, leaving the current one clean. Same flexible naming as `/worktree:add`.

- **`/worktree:status`**
  Dashboard of every worktree — folder, branch, upstream, dirty state, and
  ahead/behind counts. Read-only.

- **`/worktree:sync`**
  Fetch the remote once, then fast-forward every clean, behind-only worktree.

### Cleanup

- **`/worktree:remove <folder-name>`**
  Remove a single worktree from the repo and prune.

- **`/worktree:prune [base-branch]`**
  Prune stale worktree entries and remove worktrees whose branches are already
  merged (with confirmation).

### Help

- **`/worktree:help`**
  Show this reference.

_Args in `<angle brackets>` are required; `[square brackets]` are optional._
