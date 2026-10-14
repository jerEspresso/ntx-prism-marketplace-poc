---
description: Spin the current worktree's in-progress work onto a new branch (name, work-item #, or description)
argument-hint: "[new-branch | work-item# | description] [folder-name]"
allowed-tools: Bash, PowerShell, Read, mcp__azure-devops__wit_get_work_item, mcp__azure-devops__search_workitem
---

The user is inside a worktree (e.g. `develop`) and has started work that should
retrospectively live on its own branch. Move that work — whether uncommitted,
committed, or both — to a NEW worktree on a NEW branch, and leave the current
worktree clean (back at its upstream).

Input — `$ARGUMENTS` is flexible. It may be:
- an explicit new branch name → use as-is
- a work item number (`537744`) → derive the branch name (step 1)
- a free-text description of the change → derive the branch name (step 1)
- empty → infer from the current conversation / the work in progress; if there
  is nothing to go on, ask

An explicit trailing folder name (or "in folder X") overrides the derived
folder; otherwise the folder is the branch name with `/` -> `-`.

Use PowerShell. Because git stashes are shared across all worktrees in the repo,
uncommitted work moves cleanly between them.

Steps — run in order; STOP and report on any failure:

1. Resolve the new branch name `$1` and preconditions:
   - Current directory must be inside a worktree of a bare + worktree repo.
     Record the current worktree path (`$here`), the current branch (`$cur`),
     and the container root (the parent holding `.bare` + the `.git` file).
   - If the input is not already a branch name, derive `$1` per the **Branch /
     worktree naming** convention in `X:\CLAUDE.md` (format
     `<type>/<item>-<kebab-desc>`; for a work item number fetch its title from
     Azure DevOps and map its type). Auto-name when unambiguous; confirm only if
     genuinely unsure. This is always a NEW branch.
   - If a branch named `$1` already exists → STOP.
   - Determine the target folder (explicit override else `$1` with `/`->`-`) and
     its full path under the container root; if it already exists → STOP.

2. Assess what needs moving:
   - Dirty? `git status --porcelain` — non-empty means uncommitted changes.
   - Upstream: `$up = git rev-parse --abbrev-ref --symbolic-full-name '@{u}' 2>$null`.
   - Local commits ahead of upstream: if `$up` exists,
     `git rev-list --count '@{u}..HEAD'`; else unknown.
   - If NOT dirty AND not ahead of upstream → there is nothing to spin off;
     report that and STOP.

3. If dirty, stash everything (including untracked) so the working tree is clean:
   ```powershell
   git stash push -u -m "wt-branch:$1"
   ```
   Remember that a stash was created.

4. Create the new branch at the current commit (captures any local commits):
   ```powershell
   git branch $1 HEAD
   ```

5. Rewind the current branch ONLY if it was ahead of upstream (i.e. there were
   local commits to move off it):
   - If ahead and `$up` exists — this REWRITES `$cur`, so first show the user
     exactly what will happen (current branch, from-SHA → upstream-SHA, N
     commits being moved to `$1`) and CONFIRM before running:
     ```powershell
     git reset --hard '@{u}'
     ```
   - If ahead but there is NO upstream on `$cur` → STOP and ask the user which
     base ref `$cur` should be reset to; do not guess.
   - If not ahead (only uncommitted work) → skip this step entirely; `$cur`
     already matches upstream once the stash is taken.

6. From the container root, create the new worktree checked out on `$1`:
   ```powershell
   git worktree add <folder> $1
   ```

7. If a stash was created in step 3, apply it in the NEW worktree:
   ```powershell
   git -C <folder> stash pop
   ```
   If pop reports conflicts, STOP and tell the user (the stash is preserved).

8. Report: the new worktree path + branch (and, if you derived the name, what
   you named it and why), confirm `$cur` is now clean/at upstream, and note that
   `$1` will need `git push -u origin $1` on first push.

9. Ask the user whether to switch this session to the new worktree directory.
   If they say yes, `cd` into `<folder>` so subsequent commands in this session
   run there. If no, stay put and just remind them the work now lives in that
   folder. Note that this only moves THIS session's working directory — it does
   not affect any other open terminal or Claude session.
