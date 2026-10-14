# ntx-prism-marketplace-poc

A [Claude Code plugin marketplace](https://code.claude.com/docs/en/plugins/create-marketplace) for NTX Prism (proof of concept).

## Install plugins

Add the marketplace once, then install plugins from it as `<plugin>@ntx-prism-marketplace-poc`:

```bash
claude plugin marketplace add jerEspresso/ntx-prism-marketplace-poc
claude plugin install worktree@ntx-prism-marketplace-poc
```

Inside a Claude Code session, use `/plugin marketplace add jerEspresso/ntx-prism-marketplace-poc` and `/plugin install worktree@ntx-prism-marketplace-poc` instead.

This repository is private. Claude Code clones it with the git credentials already on your machine and can't prompt for them, so you need read access plus either a GitHub SSH key loaded in `ssh-agent` or stored HTTPS credentials (`gh auth login`, then `gh auth setup-git`).

To pick up changes, run `claude plugin update <plugin>@ntx-prism-marketplace-poc`, or turn on auto-update for this marketplace under **Marketplaces** in `/plugin`.

## Plugins

| Plugin | Description |
| :- | :- |
| [`worktree`](plugins/worktree) | Work with bare + worktree git repos: clone, convert, add, branch, status, sync, remove, and prune worktrees. Run `/worktree:help` for the command list. |

## Add a plugin

```text
.claude-plugin/marketplace.json    catalog: marketplace name, owner, one entry per plugin
plugins/<name>/
  .claude-plugin/plugin.json       plugin manifest (the only file inside .claude-plugin/)
  skills/<skill>/SKILL.md          components go at the plugin root
```

1. Create the plugin under `plugins/<name>/`.
2. Add an entry to `plugins` in `.claude-plugin/marketplace.json`:
   - `name` must match the `name` in the plugin's `plugin.json`.
   - `source` is a path from the repository root, such as `./plugins/<name>`, with no `..`.
3. Run `claude plugin validate .` and fix anything it reports.

If a plugin's `plugin.json` sets `version`, bump it with every change you want users to receive; they stay on their cached copy until it changes. Omit `version` to have users track commits instead.

## Test locally

Register your clone instead of the GitHub repository. Claude Code then reads plugins straight from your working tree, so edits apply at the next session start or after `/reload-plugins`:

```bash
claude plugin marketplace add ./
claude plugin install worktree@ntx-prism-marketplace-poc
```

A marketplace name can be registered only once per user, so remove the GitHub-registered copy first. `claude plugin marketplace remove ntx-prism-marketplace-poc` removes the marketplace and uninstalls its plugins.
