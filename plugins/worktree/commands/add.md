---
description: Add a worktree to a bare+worktree repo (branch name, work-item #, or description)
argument-hint: "[branch | work-item# | description] [folder-name]"
allowed-tools: Bash, PowerShell, Read, mcp__azure-devops__wit_get_work_item, mcp__azure-devops__search_workitem
---

Add a new worktree to a bare + worktree repo (one created by `/worktree:clone`) so a
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

7. Create an empty `.env` file in the root of the new worktree. `.env` is
   normally gitignored, so `git worktree add` never brings one along. If the
   file already exists (the branch tracks one), leave it untouched:
   ```powershell
   $dotEnv = Join-Path $path '.env'
   if (-not (Test-Path $dotEnv)) { New-Item -ItemType File -Path $dotEnv | Out-Null }
   ```

8. Install dependencies if the new worktree contains `package.json`; skip this
   step otherwise. This runs before step 9 so the new worktree has
   `playwright-webkit` for the token script.
   ```powershell
   if (Test-Path (Join-Path $path 'package.json')) {
     Push-Location $path; npm install; Pop-Location
   }
   ```
   Use plain `npm install`; the repo's own `.npmrc` adds any flags it needs
   (ntx-tables-client's sets `legacy-peer-deps=true`). It can take several
   minutes, so give it a long timeout (up to 10 minutes). If it fails, show
   the error (an `E401` from a private registry usually means the npm
   credentials need refreshing), then carry on with step 9 instead of stopping.
   The worktree already exists, and step 9 can still copy a token or run the
   script from another worktree.

9. Share the automation refresh token. Run this step only for the
   `ntx-tables-client` repo; skip it for any other repo. Get the repo name from
   the end of the `origin` URL, because the container folder can have any name:
   ```powershell
   $repoName = ((git remote get-url origin) -split '[/:]')[-1] -replace '\.git$', ''
   ```
   If the new branch predates `.env.automation.test` (the new worktree has no
   such file), say so in the report and skip the rest of this step. Copying
   the file in would leave it untracked, and a later `git add .` could commit
   the token.
   - A file counts as fresh if it was written in the last 20 hours **and**
     contains an active `AUTOMATION_REFRESH_TOKEN=<value>` line. Checking the
     content matters because `git worktree add` gives every file a new write
     time, even when it still holds the committed baseline with no token. Look
     for a fresh copy in the other worktrees:
     ```powershell
     $envName = '.env.automation.test'
     $newEnv  = Join-Path $path $envName
     $self    = [IO.Path]::GetFullPath($path).TrimEnd('\')
     $others  = git worktree list --porcelain |
       Where-Object { $_ -like 'worktree *' } |
       ForEach-Object { [IO.Path]::GetFullPath($_.Substring(9)).TrimEnd('\') } |
       Where-Object { $_ -ne $self -and (Test-Path (Join-Path $_ $envName)) }
     $fresh = $others | ForEach-Object { Get-Item (Join-Path $_ $envName) } |
       Where-Object { $_.LastWriteTime -gt (Get-Date).AddHours(-20) -and
         (Select-String -LiteralPath $_.FullName -Pattern '^AUTOMATION_REFRESH_TOKEN=\S' -Quiet) } |
       Sort-Object LastWriteTime -Descending | Select-Object -First 1
     ```
   - **If `$fresh` is set**, copy it to the new worktree:
     `Copy-Item $fresh.FullName $newEnv -Force`. `Copy-Item` keeps the source's
     write time, so the 20-hour window still counts from when the token was
     issued.
   - **If not**, get a new token with the `get-refresh-token` skill's script.
     That skill has `disable-model-invocation`, so run the script directly
     instead of through the Skill tool. Point the script at a checkout that has
     `playwright-webkit`. After step 8 that is normally the new worktree; any
     other worktree with its packages installed works too. If no checkout has
     it (step 8 failed and no other worktree has run `npm install`), STOP this
     step and report it.
     ```powershell
     $runner = git worktree list --porcelain |
       Where-Object { $_ -like 'worktree *' } | ForEach-Object { $_.Substring(9) } |
       Where-Object { Test-Path (Join-Path $_ 'node_modules\playwright-webkit') } |
       Select-Object -First 1
     $env:NTX_TABLES_CLIENT_REPO = $runner
     node "C:\Users\XiaoJ\.claude\skills\get-refresh-token\scripts\get-refresh-token.mjs"
     ```
     Confirm the JSON on stdout has `"ok": true`, and check that the new
     worktree's file appears in `env_files`. On failure, report it and follow
     the skill's Troubleshooting section. Don't copy anything afterwards. The
     script already writes the token into every worktree's own file and edits
     only the token lines, so each branch's other changes to the file stay
     intact.

10. Report the new worktree's full path (and, if you derived the name, what you
    named it and why) and remind the user to point their Claude Code / VS Code
    session at that subfolder. Say whether step 7 created `.env` or found one
    already there, and whether step 8's `npm install` succeeded or was
    skipped. Say how step 9 went: which worktree the token was copied from
    and how old it is, or that a new token was issued and which worktrees
    received it (the script's `env_files`, plus any `skipped_checkouts` or
    `failed`). Remind the user that `.env.automation.test` is tracked by git
    and the token change must never be committed.

11. Ask the user whether to switch this session to the new worktree directory.
    If yes, `cd` into `<folder>` so subsequent commands in this session run
    there. If no, stay put. Note this only moves THIS session's working
    directory — it does not affect other terminals or Claude sessions.
