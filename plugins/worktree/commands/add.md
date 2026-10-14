---
description: Add a worktree to a bare+worktree repo (branch name, work-item #, or description)
argument-hint: "[branch | work-item# | description] [folder-name]"
allowed-tools: Bash, PowerShell, Read, mcp__azure-devops__wit_get_work_item, mcp__azure-devops__search_workitem
---

Add a new worktree to a bare + worktree repo (one created by `/wt:clone`) so a
branch can be checked out in parallel with the others.

Input — `$ARGUMENTS` is flexible. It may be:
- an explicit branch name (`fix/537744-foo`, or an existing branch) → use as-is
- a work item number (`537744`) → derive the branch name (step 3)
- a free-text description of the change → derive the branch name (step 3)
- empty → infer the intent from the current conversation; if there is nothing
  to go on, ask

An explicit trailing folder name (or "in folder X") overrides the derived
folder; otherwise the folder is the branch name with `/` -> `-`.

Use PowerShell.

Steps — run in order, stop and report on any failure:

1. Confirm the current directory is a bare + worktree container root — it should
   contain a `.bare` directory and a `.git` **file**. If not, check whether the
   user is inside a worktree subfolder and `cd` up to the container root; if it
   is not this layout at all, STOP and explain.

2. Fetch first so branch existence checks are accurate:
   ```powershell
   git fetch origin
   ```

3. Resolve the branch name `$1`:
   - If the input already IS a branch name (contains `/`, or matches an existing
     local/remote branch) → use it verbatim.
   - Otherwise derive it per the **Branch / worktree naming** convention in
     `X:\CLAUDE.md` (format `<type>/<item>-<kebab-desc>`; for a work item number
     fetch its title from Azure DevOps and map its type). This is a NEW branch.
   - Auto-create when the resulting name is unambiguous; pause to confirm only
     if you are genuinely unsure or the request was too vague to name.

4. Determine the folder name (explicit override, else `$1` with `/` -> `-`) and
   set `$path = <container-root>\<folder>`. If `$path` already exists, STOP.

5. Decide new-vs-existing branch and add accordingly:
   - If a local or remote branch named `$1` already exists, check it out:
     ```powershell
     git worktree add <folder> $1
     ```
   - Otherwise create a new branch off the current default/HEAD:
     ```powershell
     git worktree add <folder> -b $1
     ```
   Note: a branch already checked out in another worktree cannot be added
   again — if git refuses for that reason, report which worktree holds it
   (`git worktree list`) rather than forcing it.

6. Wire up upstream tracking. `git worktree add` only auto-tracks (DWIM) when
   it has to CREATE a local branch from a remote-only branch — and in a bare
   layout `git clone --bare` already imported every origin branch as a local
   head, so that almost never fires. Set it explicitly so ahead/behind and
   `git pull`/`push` work:
   ```powershell
   git ls-remote --exit-code --heads origin $1 *> $null
   if ($LASTEXITCODE -eq 0) {
     git -C <folder> branch --set-upstream-to=origin/$1 $1
   }
   ```
   (If no matching remote branch exists — a brand-new local branch — skip this;
   the upstream gets set later by `git push -u origin $1`.)

7. Report the new worktree's full path (and, if you derived the name, what you
   named it and why) and remind the user to point their Claude Code / VS Code
   session at that subfolder.

8. Ask the user whether to switch this session to the new worktree directory.
   If yes, `cd` into `<folder>` so subsequent commands in this session run
   there. If no, stay put. Note this only moves THIS session's working
   directory — it does not affect other terminals or Claude sessions.
