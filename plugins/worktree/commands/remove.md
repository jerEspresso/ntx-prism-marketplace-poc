---
description: Remove a worktree from a bare+worktree repo and prune
argument-hint: <folder-name>
allowed-tools: Bash, PowerShell, Read
---

Remove a worktree (created by `/wt:clone` or `/wt:add`) and clean up.

Arguments:
- `$1` = worktree folder name to remove (required)

Use PowerShell.

Steps — run in order, stop and report on any failure:

1. Confirm the current directory is a bare + worktree container root (has a
   `.bare` directory and a `.git` file). If the user is inside the worktree
   they're trying to remove, `cd` up to the container root first — you cannot
   remove the worktree you're standing in.

2. Show what will be removed so the user can sanity-check:
   ```powershell
   git worktree list
   ```

3. Attempt a clean removal:
   ```powershell
   git worktree remove $1
   ```

4. If git refuses because the worktree has uncommitted changes or is locked,
   do NOT silently force. STOP and report exactly what's dirty/locked
   (`git -C $1 status --short`) and ask the user to confirm before retrying
   with `git worktree remove --force $1`.

5. Prune any stale administrative entries:
   ```powershell
   git worktree prune
   ```

6. Report which worktree was removed and show the remaining ones
   (`git worktree list`).
