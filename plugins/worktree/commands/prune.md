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
   - **clean** — no uncommitted changes, **or** the only uncommitted change is
     `.env.automation.test` (a locally refreshed test token, safe to discard),
   - has an upstream configured whose remote branch no longer exists ("gone").

   Check cleanliness per worktree (`--untracked-files=all` so a nested
   untracked file can't hide behind its folder):
   ```powershell
   $dirty = @(git -C <folder> status --porcelain --untracked-files=all)
   $other = @($dirty | Where-Object { $_.Substring(3) -ne '.env.automation.test' })
   # $dirty empty            → clean
   # $dirty set, $other empty → eligible, but .env.automation.test must be discarded first
   # $other set               → dirty, skip
   ```

   Detect gone branches (upstream configured AND its remote ref deleted):
   ```powershell
   $gone = git for-each-ref --format '%(refname:short)|%(upstream)|%(upstream:track)' refs/heads |
     ForEach-Object { $p = $_ -split '\|',3; if ($p[1] -and $p[2] -eq '[gone]') { $p[0] } }
   ```
   An empty `%(upstream)` means no remote is configured → skip (never touch).
   `%(upstream:track)` of `[gone]` means the configured remote branch was
   deleted → candidate. Cross-reference `$gone` with the worktree list, then
   drop `develop`/`master`, detached, and dirty worktrees.

   For each remaining candidate, check whether its branch is merged into
   `develop`: either its tip is in `origin/develop`'s history (regular merge),
   or its net change since the merge-base matches one commit on
   `origin/develop` by patch-id (squash-merged PR):
   ```powershell
   function Get-DevelopMerge($branch) {
     git merge-base --is-ancestor $branch origin/develop
     if ($LASTEXITCODE -eq 0) { return 'merged' }
     $mb = git merge-base $branch origin/develop
     $branchId = ((git diff $mb $branch | git patch-id --stable) -split ' ')[0]
     if (-not $branchId) { return $null }
     $hit = git log -p --no-merges "$mb..origin/develop" | git patch-id --stable |
       Where-Object { ($_ -split ' ')[0] -eq $branchId } | Select-Object -First 1
     if ($hit) { return 'squash-merged as ' + (git log -1 '--format=%h %s' ($hit -split ' ')[1]) }
     return $null
   }
   # non-null → merged into develop; $null → NOT merged
   ```
   A squash commit that was changed on completion (conflict fix, extra edit)
   no longer matches and comes back NOT merged. That is the safe direction: it
   falls to the user.

5. If there are no candidates, report that and stop. Otherwise LIST the
   candidates (folder + branch + the gone upstream + merge status: `merged`,
   `squash-merged as <sha> <subject>`, or `NOT merged`) and STOP to ask the
   user to confirm before removing anything — do not auto-delete. Say that
   merged branches will be force-deleted (`-D`), and flag every candidate whose
   `.env.automation.test` changes will be discarded, so the confirmation covers
   both.

6. On confirmation, for each approved candidate:
   ```powershell
   # Only for candidates flagged in step 5 — discard the .env.automation.test change:
   git -C <folder> restore --source=HEAD --staged --worktree -- .env.automation.test  # tracked file
   Remove-Item <folder>\.env.automation.test                                          # untracked (`??`) file
   # Re-run the step 4 status check; if anything is still dirty, skip this candidate.

   git worktree remove <folder>
   git branch -D <branch>        # merged into develop (step 4): its work is on develop
   git branch -d <branch>        # NOT merged: -d only; refuses if the branch is unmerged
   ```
   Force-delete (`-D`) only branches that step 4 found merged into `develop`.
   A squash merge leaves the branch's own commits outside `develop`'s history,
   so `-d` would refuse even though the work landed. For a branch that is NOT
   merged, use `-d`. If git refuses, do **NOT** force: report it and offer
   `git branch -D <branch>` as an explicit follow-up the user can approve.

7. Report what was removed and what was skipped (and why), then show the
   remaining worktrees (`git worktree list`). For every force-deleted branch,
   include the SHA git printed (`was <sha>`) so it can be recovered with
   `git branch <name> <sha>`.
