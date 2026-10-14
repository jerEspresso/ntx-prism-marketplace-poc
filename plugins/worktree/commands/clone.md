---
description: Clone a repo into the bare + worktree layout under X:\repos
argument-hint: <git-url> [folder-name]
allowed-tools: Bash, PowerShell, Read
---

Clone the repository at `$1` using the **bare + worktree layout** so it can be
checked out on multiple branches simultaneously for parallel sessions.

Arguments:
- `$1` = git remote URL (required)
- `$2` = container folder name (optional; if omitted, derive from the URL by
  taking the last path segment and stripping a trailing `.git`)

Base directory is `X:\repos`. Use PowerShell.

Steps — run them in order, stopping and reporting if any step fails:

1. Derive the container name (from `$2`, else from `$1`) and set
   `$dir = X:\repos\<name>`. If `$dir` already exists, STOP and tell the
   user rather than overwriting.

2. Create and enter the container:
   ```powershell
   mkdir $dir; cd $dir
   ```

3. Bare-clone the history once into `.bare`:
   ```powershell
   git clone --bare "$1" .bare
   ```

4. Add the `.git` pointer file so plain `git` works from the container root
   (no file extension):
   ```powershell
   "gitdir: ./.bare" | Set-Content .git
   ```

5. Fix the fetch refspec (a bare clone does NOT set up remote tracking like a
   normal clone — without this, `origin/*` refs and worktree branch tracking
   misbehave), then fetch:
   ```powershell
   git config remote.origin.fetch "+refs/heads/*:refs/remotes/origin/*"
   git fetch origin
   ```

6. Detect the repo's default branch:
   ```powershell
   $default = (git -C .bare symbolic-ref --short HEAD)
   ```

7. Add a worktree for the default branch and wire up its upstream tracking.
   The bare clone imported the default branch as a plain local head with no
   tracking config, and `git worktree add` won't set it for a pre-existing
   local branch — so set it explicitly:
   ```powershell
   git worktree add $default
   git -C $default branch --set-upstream-to=origin/$default $default
   ```

8. Optionally hide the plumbing from Explorer:
   ```powershell
   attrib +h "$dir\.bare" "$dir\.git"
   ```

9. Report the result: the container path, the default-branch worktree path
   (`$dir\<default>`), and remind the user they can add more parallel checkouts
   with `/worktree:add`. Tell them to point each Claude Code / VS Code session at a
   worktree subfolder, never the container root.
