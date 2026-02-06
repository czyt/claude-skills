---
name: agent-worktree
description: "Git worktree workflow tool for AI coding agents. Use when managing parallel development environments, creating isolated worktrees, running snap mode workflows, merging branches, or any task involving the `wt` command. Triggers on: worktree creation, branch isolation, parallel agent execution, snap mode, wt commands."
---

# agent-worktree: Git Worktree Workflow for AI Agents

`agent-worktree` (`wt`) provides isolated parallel development environments via Git worktrees. It is designed for AI coding agents that work best in isolated branches.

## Core Concepts

- **Worktree**: An isolated working directory linked to a Git branch, stored under `~/.agent-worktree/workspaces/{project}/`
- **Trunk**: The main branch (auto-detected or configured, typically `main` or `master`)
- **Snap Mode**: A one-shot workflow — create worktree, run agent, review changes, merge, clean up

## Prerequisites

Install globally via npm:

```bash
npm install -g agent-worktree
```

Run `wt setup` to install shell integration (supports bash, zsh, fish, PowerShell). Update with `wt update`.

## Command Reference

### Worktree Management

| Command | Description |
|---------|-------------|
| `wt new [branch]` | Create worktree (random name if branch omitted) |
| `wt new --base <ref>` | Create worktree based on a specific commit/branch |
| `wt new -s <cmd>` | Create worktree + enter snap mode with agent command |
| `wt cd <branch>` | Switch to a worktree |
| `wt ls` | List all worktrees |
| `wt main` | Return to main repository |
| `wt mv <old> <new>` | Rename worktree (`.` = current) |
| `wt rm <branch>` | Delete worktree (`.` = current) |
| `wt rm -f <branch>` | Force delete (including uncommitted changes) |
| `wt clean` | Clean up worktrees with no diff from trunk |

### Workflow Commands

| Command | Description |
|---------|-------------|
| `wt merge` | Merge current worktree into trunk |
| `wt merge -s <strategy>` | Merge with strategy: `squash`, `merge`, or `rebase` |
| `wt merge --into <branch>` | Merge into a specific branch instead of trunk |
| `wt merge -k` | Merge but keep the worktree |
| `wt merge --continue` | Continue merge after resolving conflicts |
| `wt merge --abort` | Abort an in-progress merge |
| `wt sync` | Sync current worktree from trunk (rebase) |
| `wt sync -s merge` | Sync using merge strategy |
| `wt sync --continue` | Continue sync after resolving conflicts |
| `wt sync --abort` | Abort an in-progress sync |

### Configuration Commands

| Command | Description |
|---------|-------------|
| `wt setup` | Install shell integration (auto-detect shell) |
| `wt setup --shell zsh` | Install for a specific shell |
| `wt init` | Initialize project-level config |
| `wt init --trunk <branch>` | Initialize with explicit trunk branch |

## Snap Mode

Snap mode is the primary workflow for AI agent tasks. It automates: create worktree -> enter -> run agent -> review -> merge -> clean up.

```bash
# Random branch name, run claude
wt new -s claude

# Named branch, run cursor
wt new fix-bug -s cursor

# Agent with arguments
wt new -s "aider --model sonnet"
```

When the agent exits:
- **No changes**: Worktree is auto-cleaned
- **Only commits (no uncommitted changes)**: Prompt to `[m]` merge or `[q]` quit
- **Uncommitted changes exist**: Prompt to `[r]` reopen agent or `[q]` quit (handle manually)

## Configuration

### Global config: `~/.agent-worktree/config.toml`

```toml
[general]
merge_strategy = "squash"  # squash | merge | rebase
trunk = "main"             # trunk branch (auto-detected if omitted)
copy_files = [".env", ".env.*"]  # gitignore-style patterns to copy into new worktrees

[hooks]
post_create = ["pnpm install"]
pre_merge = ["pnpm test", "pnpm lint"]
post_merge = []
```

### Project config: `.agent-worktree.toml`

```toml
[general]
copy_files = ["*.secret.*"]
```

Project config merges with global config. Place `.agent-worktree.toml` in the repository root.

## Storage Layout

```
~/.agent-worktree/
├── config.toml                    # Global config
└── workspaces/
    └── {project}/
        ├── swift-fox.status.toml  # Worktree metadata
        ├── swift-fox/             # Worktree directory
        └── ...
```

## Usage Guidelines

When helping users with `wt` commands:

1. **Creating worktrees**: Default to `wt new <descriptive-branch-name>` for clarity. Use random names only for throwaway work.
2. **Merging**: Default merge strategy is `squash` for clean history. Use `--into` when targeting non-trunk branches.
3. **Conflict resolution**: Guide users through `wt merge --continue` or `wt sync --continue` after manual conflict resolution.
4. **Snap mode**: Recommend for one-off agent tasks. The agent command should be a single executable or quoted command string.
5. **Cleanup**: Suggest `wt clean` periodically to remove stale worktrees that have already been merged.
6. **Environment files**: Remind users to configure `copy_files` in config if they rely on `.env` or similar files not tracked by Git.
7. **Hooks**: Suggest `post_create` hooks for dependency installation (e.g., `pnpm install`) to ensure worktrees are ready to use immediately.
