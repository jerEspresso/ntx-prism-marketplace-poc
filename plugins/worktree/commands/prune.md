---
description: Prune stale worktree entries and remove worktrees whose remote branch is gone (with confirmation)
allowed-tools: Bash, PowerShell, Read
---

Clean up the current bare + worktree repo: remove git's stale administrative
entries, then find worktrees whose configured upstream branch has been **deleted
on the remote** ("gone") and offer to remove them.

Only remove a worktree/branch when it has a remote tracking branch configured
AND that remote branch no longer exists. Hard rules that override everything:

- **`develop` and `master` are never candidates**, under any circumstances.
- **Branches with no upstream configured are never touched** — a missing remote
  is not the same as a deleted remote.

Use PowerShell.

Steps:

1. Confirm the current directory is a bare + worktree container root (has a
   `.bare` directory and a `.git` file); `cd` up from a worktree subfolder if
   needed. If not this layout, STOP.

2. Prune stale administrative entries (worktrees whose folder was deleted
   manually). This is always safe:
   ```powershell
   git worktree prune -v
   ```

3. Fetch and prune deleted remote-tracking refs so "gone" status is accurate:
   ```powershell
   git fetch origin --prune
   ```

4. Find removal candidates — worktrees that are ALL of:
   - branch is **not** `develop` and **not** `master`,
   - not detached,
   - **clean** (no uncommitted changes),
   - has an upstream configured whose remote branch no longer exists ("gone").

   Detect gone branches (upstream configured AND its remote ref deleted):
   ```powershell
   $gone = git for-each-ref --format '%(refname:short)|%(upstream)|%(upstream:track)' refs/heads |
     ForEach-Object { $p = $_ -split '\|',3; if ($p[1] -and $p[2] -eq '[gone]') { $p[0] } }
   ```
   An empty `%(upstream)` means no remote is configured → skip (never touch).
   `%(upstream:track)` of `[gone]` means the configured remote branch was
   deleted → candidate. Cross-reference `$gone` with the worktree list, then
   drop `develop`/`master`, detached, and dirty worktrees.

5. If there are no candidates, report that and stop. Otherwise LIST the
   candidates (folder + branch + the gone upstream) and STOP to ask the user to
   confirm before removing anything — do not auto-delete.

6. On confirmation, for each approved candidate:
   ```powershell
   git worktree remove <folder>
   git branch -d <branch>        # -d only; refuses if the branch is unmerged
   ```
   Use `-d` first. If git refuses because the branch has unmerged commits
   (common when the PR was squash-merged before the remote branch was deleted),
   do **NOT** auto-force. Report it and offer `git branch -D <branch>` as an
   explicit follow-up the user can approve — never force-delete on your own.

7. Report what was removed and what was skipped (and why), then show the
   remaining worktrees (`git worktree list`).
