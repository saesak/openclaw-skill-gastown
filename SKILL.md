---
name: gastown
description: Multi-agent coding orchestrator using Gas Town (gt) and Claude Code. Use for ANY non-trivial coding task — multi-file changes, new features, refactors, bug fixes, anything involving code that needs to compile/run/test. Delegates work to parallel Claude Code agents (polecats) with git-backed persistent state, work tracking (beads), and coordination. Use when a task involves more than a single file edit or quick script.
metadata:
  openclaw:
    emoji: "🏭"
    requires:
      allBins: ["tmux", "claude"]
    setup: "scripts/setup.sh"
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

# Create workspace
gt install ~/gt --git
cd ~/gt

# Add a project as a rig
gt rig add <name> <git-repo-or-local-path> --branch main

# Link formulas so polecats get the mol-polecat-work lifecycle (see Formula Resolution below)
cd ~/gt/<name>/.beads && ln -s ../../.beads/formulas formulas && cd ~/gt

# Fix any config issues
gt doctor --fix

# Bring up all services (Mayor, Witness, Refinery, Deacon, Daemon)
gt up
```

## Core Workflow — Work Through the Mayor

**The Mayor is your primary interface.** Don't manually create beads and sling them — the Mayor handles bead creation, formula application, convoy coordination, and merge queue orchestration.

### 1. Start services and send work to Mayor

```bash
export PATH=$PATH:$HOME/local/go/bin:$HOME/go/bin
cd ~/gt

# Make sure services are running
gt up

# Send Mayor a task description via mail
gt mail send mayor -s "Subject" -m "Description of what needs to be done"

# Then nudge Mayor to check its inbox
gt nudge mayor "You have new mail, please check inbox and dispatch."

# OR: Interactive session (attach to Mayor's tmux)
gt mayor attach
# Then describe the task in natural language.
```

**Note:** `gt mayor mail` does NOT exist. Use `gt mail send mayor` for async messages.

The Mayor will:
- Break the task into beads (work items)
- Create a convoy to track them
- Sling beads to polecats with the proper `mol-polecat-work` formula
- **Send SWARM_START to Witness** for batch completion tracking
- Monitor progress and handle coordination

### Batch Work Notification (SWARM_START)

When Mayor dispatches multiple beads as a batch, it **must** notify Witness so completion can be tracked:

```bash
gt mail send <rig>/witness -s "SWARM_START" -m '{"swarm_id": "batch-123", "beads": ["vt-abc", "vt-def", "vt-ghi"]}'
```

This triggers:
1. Witness creates a swarm tracking wisp
2. Witness monitors polecat completion each patrol cycle  
3. When all polecats in the swarm complete → Witness sends `SWARM_COMPLETE` to Mayor
4. Mayor can then dispatch dependent work

**Without SWARM_START**, Mayor has no way to know when batch work completes. It will sit idle waiting for dependencies that nobody notifies it about.

### 2. Monitor progress

```bash
# List convoys (work bundles)
gt convoy list

# Check convoy detail
gt convoy status <convoy-id>

# List all agents (including polecats)
gt agents list --all

# Peek at a polecat's current output
tmux capture-pane -t gt-<rig>-<polecat-name> -p | tail -30

# List all Gastown tmux sessions
tmux list-sessions | grep gt-

# Check bead status
bd show <bead-id>

# Check overall town health
gt status
gt doctor
```

### 3. Service Management

```bash
# Start all services
gt up

# Stop all services
gt down

# Check Mayor status
gt mayor status

# Start/restart Mayor
gt mayor start
gt mayor restart

# Check what's running
tmux list-sessions | grep gt-
```

### 4. How Polecats Work

When Mayor slings a bead, polecats follow the `mol-polecat-work` lifecycle (9 steps):
1. **load-context** — Read the bead, run `gt prime` and `bd prime`
2. **branch-setup** — Create a working branch
3. **preflight-tests** — Verify tests pass on main
4. **implement** — Do the actual work
5. **self-review** — Review own changes
6. **run-tests** — Run tests, verify coverage
7. **cleanup-workspace** — Clean up
8. **prepare-for-review** — Prepare for merge
9. **submit-and-exit** — Push to merge queue, self-destruct

Each step is tracked as a sub-bead. Polecats use `bd ready` to find their next step. Without the formula (e.g., if slung with `--hook-raw-bead`), polecats get a one-shot prompt and will idle/freeze once their initial burst of work is done — there's nothing driving them to the next step.

The **Refinery** agent merges polecat branches back to main. You never push directly.

## Anti-Patterns (Don't Do This)

| ❌ Don't | ✅ Do Instead |
|---|---|
| `bd create` + `gt sling` manually | Tell Mayor via `gt mail send mayor` or `gt mayor attach` |
| `gt sling --hook-raw-bead` | Let Mayor apply `mol-polecat-work` formula automatically |
| `gt mayor mail "..."` | Use `gt mail send mayor -s "subject" -m "message"` |
| Push to main directly | Let Refinery merge from the merge queue |
| Close beads manually | Polecats self-clean; Refinery closes after merge |
| Create polecats without Mayor | Mayor handles spawning and assignment |
| Skip `gt up` before sending work | Services must be running for Mayor to dispatch |

## Quick Reference

| Action | Command |
|---|---|
| Start all services | `gt up` |
| Stop all services | `gt down` |
| Send Mayor a task | `gt mail send mayor -s "subject" -m "description"` |
| Nudge Mayor | `gt nudge mayor "message"` |
| Attach to Mayor (interactive) | `gt mayor attach` |
| Mayor status | `gt mayor status` |
| List convoys | `gt convoy list` |
| Convoy detail | `gt convoy status <id>` |
| List agents | `gt agents list --all` |
| Peek at polecat | `tmux capture-pane -t gt-<rig>-<name> -p \| tail -30` |
| List tmux sessions | `tmux list-sessions \| grep gt-` |
| Check bead status | `bd show <bead-id>` |
| Town health | `gt doctor` |
| Full reset | `gt down && rm -rf ~/gt && gt install ~/gt --git` |

## Architecture

See `references/architecture.md` for full details on Mayor, Rigs, Polecats, Hooks, Convoys, Beads, Refinery, and Witness.

## Formula Resolution (Important)

Gastown has two tools that deal with formulas differently:

- **`gt`** (orchestrator) searches 3 paths: `.beads/formulas/` (project), `~/.beads/formulas/` (user), `$GT_ROOT/.beads/formulas/` (town)
- **`bd`** (issue tracker) only searches `.beads/formulas/` relative to the current project root

When `gt sling` assigns work to a polecat, it calls `bd cook` to instantiate the `mol-polecat-work` formula. But `bd cook` runs in the rig's directory context (e.g., `~/gt/<rig>/`), so it only looks at `~/gt/<rig>/.beads/formulas/` — which doesn't exist by default for new rigs.

The formulas are installed at the town level (`~/gt/.beads/formulas/`), and `gt formula list` finds them fine. But `gt sling` doesn't pass `--search-path` to `bd cook`, so `bd` can't find them.

**The fix:** Symlink the town-level formulas into each rig during setup (included in the Setup section above). This makes the rig's `.beads/formulas/` resolve to the shared town formulas — the intended mechanism since rigs are designed to be self-contained, and the symlink opts them into the shared formula library.

**Without this symlink**, `gt sling` logs a warning (`Could not cook formula mol-polecat-work`) and falls back to raw bead mode. Polecats without the formula get a one-shot prompt and will idle/freeze after their initial work — there are no step-beads driving them forward.

## Troubleshooting

- **Mayor not dispatching dependent work after batch completes**: Mayor didn't send `SWARM_START` to Witness. Without it, Witness doesn't track completion and never sends `SWARM_COMPLETE` back to Mayor. Nudge Mayor to check convoy status and dispatch, or manually notify: `gt mail send mayor -s "Dependencies complete" -m "vt-abc and vt-def are done, please dispatch vt-ghi"`
- **Polecat idle/frozen after initial work**: Likely slung without `mol-polecat-work` formula. Check if formulas symlink exists. Kill the polecat, verify symlink, and re-dispatch through Mayor.
- **Formula not resolving**: Symlink missing. Run `cd ~/gt/<rig>/.beads && ln -s ../../.beads/formulas formulas`. Verify: `cd ~/gt/<rig> && bd cook mol-polecat-work --dry-run`.
- **`gt mayor mail` doesn't exist**: Use `gt mail send mayor -s "subject" -m "message"` instead.
- **Mayor not responding to mail**: Check `gt mayor status`. If not running, `gt mayor start` then `gt nudge mayor "check inbox"`.
- **ICU build error on beads install**: Use `CGO_ENABLED=0 go install ...`
- **Polecat not showing in `gt agents list`**: Check tmux: `tmux list-sessions | grep gt-`
- **Need Go but no sudo**: Install to `~/local/go/` instead of `/usr/local/`
- **Config issues after setup**: Run `gt doctor --fix` to auto-repair most problems.
