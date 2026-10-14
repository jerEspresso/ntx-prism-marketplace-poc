---
description: Convert an existing normal clone in-place into the bare + worktree layout (non-destructive)
argument-hint: [path]
allowed-tools: Bash, PowerShell, Read
---

Convert an existing NORMAL git clone into the bare + worktree layout, in place,
WITHOUT re-cloning (the object store — all local branches, commits, and stashes
— is preserved) and WITHOUT deleting anything (the old checkout is moved into a
backup folder the user removes once satisfied).

Arguments:
- `$1` = path to the existing clone (optional; default to the current directory).

Use PowerShell. This rearranges a real repo, so inspect and CONFIRM before
touching anything, and never force.

Steps:

1. Resolve the target `$repo` (`$1` or cwd). Validate it is a normal clone:
   - `$repo\.git` must exist AND be a **directory**. If `.git` is a file or a
     `.bare` folder already exists, it is already worktree layout → STOP.
   - If it is not a git repo at all → STOP.

2. Inspect and REPORT before doing anything (read-only):
   - remote(s): `git -C $repo remote -v`
   - current branch `$cur`: `git -C $repo rev-parse --abbrev-ref HEAD`
   - local branches: `git -C $repo branch`
   - dirty? `git -C $repo status --porcelain`
   - untracked + ignored present: `git -C $repo status --porcelain --ignored`
   - unpushed commits (exist only here): for each local branch compare to its
     upstream; warn about any that are ahead / have no upstream.

3. Present the plan and the fact that unpushed local branches/commits are kept
   (they live in the object store, not re-downloaded), then CONFIRM before
   proceeding. Do not continue without confirmation.

4. If the working tree is dirty, stash everything (kept in the object store,
   survives the move):
   ```powershell
   git -C $repo stash push -u -m "wt-convert"
   ```
   Remember that a stash was created.

5. Convert the plumbing in place:
   ```powershell
   Move-Item "$repo\.git" "$repo\.bare"
   git --git-dir="$repo\.bare" config core.bare true
   "gitdir: ./.bare" | Set-Content "$repo\.git"
   ```
   Then ensure the fetch refspec is present (normal clones usually have it; a
   bare repo may not) and fetch:
   ```powershell
   cd $repo
   if (-not (git config remote.origin.fetch)) {
     git config remote.origin.fetch "+refs/heads/*:refs/remotes/origin/*"
   }
   git fetch origin
   ```

6. Move the OLD checkout out of the container root into a backup folder — every
   top-level item EXCEPT `.bare` and the `.git` pointer (this is reversible;
   nothing is deleted):
   ```powershell
   New-Item -ItemType Directory "$repo\.pre-wt-backup" | Out-Null
   Get-ChildItem $repo -Force |
     Where-Object { $_.Name -notin '.bare', '.git', '.pre-wt-backup' } |
     ForEach-Object { Move-Item $_.FullName "$repo\.pre-wt-backup\" }
   ```

7. Recreate the current branch as a clean worktree (folder = `$cur` with
   `/` -> `-`):
   ```powershell
   git worktree add <folder> $cur
   ```
   If git says the branch is already checked out, run `git worktree prune` and
   retry; if still blocked, STOP and report rather than forcing.

8. If a stash was created in step 4, apply it in the new worktree:
   ```powershell
   git -C <folder> stash pop
   ```
   On conflict, STOP and report (the stash is preserved).

9. Offer to carry over any untracked local files the user cares about (e.g.
   `.env`, local config, `.claude/settings.local.json`) from `.pre-wt-backup`
   into the new worktree. Do NOT auto-copy build outputs / `node_modules` / `bin`
   / `obj` — those should be regenerated.

10. Verify and report:
    - `git worktree list`, `git -C <folder> status -sb`, `git -C <folder> log -1 --oneline`
    - Show the final structure (`.bare\`, `.git`, `<folder>\`, `.pre-wt-backup\`).
    - Remind the user to: re-run restore (`dotnet restore` / `npm install`)
      since build outputs weren't carried; and delete `.pre-wt-backup` once
      they've confirmed everything works (it holds the old checkout +
      untracked/ignored files as a safety net).

11. Ask the user whether to switch this session to the new worktree directory.
    If yes, `cd` into `<folder>` so subsequent commands in this session run
    there. If no, stay put. Note this only moves THIS session's working
    directory — it does not affect other terminals or Claude sessions.
