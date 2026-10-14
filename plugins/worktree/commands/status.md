---
description: Dashboard of all worktrees — branch, upstream, dirty, ahead/behind
allowed-tools: Bash, PowerShell, Read
---

Show a status dashboard across every worktree in the current bare + worktree
repo: for each worktree its folder, branch (or detached), upstream, working-tree
dirtiness, and ahead/behind counts vs its upstream.

Use PowerShell. Do NOT modify anything — this is read-only.

Steps:

1. Confirm the current directory is a bare + worktree container root (has a
   `.bare` directory and a `.git` file). If the user is inside a worktree
   subfolder, `cd` up to the container root. If it is not this layout, STOP and
   say so.

2. Enumerate worktrees and gather per-worktree detail. Suggested script:
   ```powershell
   $wts = @()
   git worktree list --porcelain | Out-String -Stream | ForEach-Object {
     if ($_ -match '^worktree (.+)$') { $path = $Matches[1] }
     elseif ($_ -match '^branch refs/heads/(.+)$') { $wts += [pscustomobject]@{ Path=$path; Branch=$Matches[1] } }
     elseif ($_ -match '^detached$') { $wts += [pscustomobject]@{ Path=$path; Branch='(detached)' } }
   }
   foreach ($w in $wts) {
     $up = (git -C $w.Path rev-parse --abbrev-ref --symbolic-full-name '@{u}' 2>$null)
     $dirty = (git -C $w.Path status --porcelain | Measure-Object -Line).Lines
     $ab = if ($up) { git -C $w.Path rev-list --left-right --count '@{u}...HEAD' 2>$null } else { $null }
     $behind, $ahead = if ($ab) { $ab -split '\s+' } else { @('-','-') }
     [pscustomobject]@{
       Folder = Split-Path $w.Path -Leaf
       Branch = $w.Branch
       Upstream = if ($up) { $up } else { '(none)' }
       Dirty  = if ($dirty -gt 0) { "$dirty" } else { 'clean' }
       Ahead  = $ahead
       Behind = $behind
     }
   }
   ```

3. Present the result as a compact table sorted by folder. Flag anything
   noteworthy in one line: worktrees with no upstream, dirty worktrees, and
   any that are behind (can be fast-forwarded via `/worktree:sync`).
