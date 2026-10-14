---
description: Fetch once, then fast-forward every clean behind-only worktree
allowed-tools: Bash, PowerShell, Read
---

Fetch the remote once (the shared object store means one fetch updates every
worktree), then report each worktree's position vs its upstream and
**fast-forward** the ones that can be advanced with zero risk.

Use PowerShell.

Steps:

1. Confirm the current directory is a bare + worktree container root (has a
   `.bare` directory and a `.git` file); `cd` up from a worktree subfolder if
   needed. If not this layout, STOP.

2. Fetch and prune deleted remote branches:
   ```powershell
   git fetch origin --prune
   ```

3. For each worktree, compute upstream + ahead/behind + dirty (same logic as
   `/worktree:status`).

4. Classify and act:
   - **behind > 0, ahead = 0, clean** → safe to fast-forward. Do it:
     ```powershell
     git -C <path> merge --ff-only '@{u}'
     ```
     Fast-forward can never lose work, so this is safe to do automatically.
   - **diverged** (behind > 0 AND ahead > 0) → do NOT touch. Report that it
     needs a manual merge/rebase.
   - **dirty** → do NOT touch. Report it has uncommitted changes.
   - **no upstream** → report and skip.
   - **up to date** → nothing to do.

5. Report a summary: which worktrees were fast-forwarded, which were skipped and
   why. Never run a non-fast-forward merge or rebase automatically.
