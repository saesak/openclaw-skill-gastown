---
name: gastown
description: Multi-agent coding orchestrator using Gas Town (gt) and Claude Code. Use for ANY non-trivial coding task — multi-file changes, new features, refactors, bug fixes, anything involving code that needs to compile/run/test. Delegates work to parallel Claude Code agents (polecats) with git-backed persistent state, work tracking (beads), and coordination. Use when a task involves more than a single file edit or quick script.
---

# Gastown — Multi-Agent Coding Orchestrator

Gastown coordinates multiple Claude Code agents to complete coding tasks in parallel with persistent state and work tracking.

## When to Use

- **Gastown**: Multi-file changes, features, refactors, bug fixes, any code needing compile/run/test
- **Sub-agents**: Quick research, one-shot generation, simple file writes, non-coding tasks
- **Direct exec**: Single file edits, running commands, checking status

## Prerequisites

- `gt` CLI (Gas Town)
- `bd` CLI (Beads issue tracker)
- `claude` CLI (Claude Code)
- `tmux` 3.0+
- Go 1.23+

Run `scripts/setup.sh` to install all prerequisites.

## Setup (first time only)

```bash
# Install gt + bd
scripts/setup.sh

# Create workspace (if not done)
gt install ~/gt --git
cd ~/gt

# Add a project as a rig
gt rig add <name> <git-repo-or-local-path> --branch main
```

## Core Workflow

### 1. Create work items (beads)

```bash
export PATH=$PATH:$HOME/local/go/bin:$HOME/go/bin
cd ~/gt
bd create --title "Description of task" --prefix <rig-prefix>
```

### 2. Create a convoy (groups related beads)

```bash
gt convoy create "Feature Name" <bead-id-1> <bead-id-2> --notify
```

### 3. Sling work to agents

```bash
gt sling <bead-id> <rig-name> --hook-raw-bead
```

Each sling spawns a **polecat** — an ephemeral Claude Code agent in a tmux session that reads the bead, does the work, commits, and reports.

### 4. Monitor progress

```bash
# List convoys
gt convoy list

# Check polecat tmux sessions
tmux list-sessions | grep gt-

# Read a polecat's current output
tmux capture-pane -t gt-<rig>-<polecat-name> -p | tail -30

# List all running agents
gt agents list
```

### 5. Review results

Polecats commit to their own branches (`polecat/<name>/<bead>@<hash>`). The refinery agent handles merging.

## Quick Reference

| Action | Command |
|---|---|
| Create bead | `bd create --title "..." --prefix <pfx>` |
| Create convoy | `gt convoy create "name" <beads...> --notify` |
| Sling to agent | `gt sling <bead> <rig> --hook-raw-bead` |
| List convoys | `gt convoy list` |
| Convoy detail | `gt convoy status <id>` |
| List agents | `gt agents list` |
| Peek at polecat | `tmux capture-pane -t gt-<rig>-<name> -p \| tail -30` |
| List tmux sessions | `tmux list-sessions \| grep gt-` |

## Architecture

See `references/architecture.md` for full details on Mayor, Rigs, Polecats, Hooks, Convoys, and Beads.

## Troubleshooting

- **`mol-polecat-work` formula not found**: Use `--hook-raw-bead` flag on `gt sling`
- **ICU build error on beads install**: Use `CGO_ENABLED=0 go install ...`
- **Polecat not showing in `gt agents list`**: Check tmux: `tmux list-sessions | grep gt-`
- **Need Go but no sudo**: Install to `~/local/go/` instead of `/usr/local/`
