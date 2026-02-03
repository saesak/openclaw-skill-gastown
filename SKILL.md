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

# Link formulas so polecats get the mol-polecat-work lifecycle
cd ~/gt/<name>/.beads && ln -s ../../.beads/formulas formulas

# Fix any config issues
gt doctor --fix
```

## Core Workflow — Work Through the Mayor

**The Mayor is your primary interface.** Don't manually create beads and sling them — the Mayor handles formula resolution, rig bootstrapping, convoy coordination, and merge queue orchestration.

### 1. Tell the Mayor what you need

```bash
export PATH=$PATH:$HOME/local/go/bin:$HOME/go/bin
cd ~/gt

# Interactive session (best for complex tasks)
gt mayor attach
# Then describe the task in natural language.
# Mayor creates beads, convoys, assigns polecats, tracks progress.

# Non-interactive (fire and forget)
gt mayor mail "Refactor the voice pipeline into a reusable library"
```

The Mayor will:
- Break the task into beads (work items)
- Create a convoy to track them
- Sling beads to polecats with the proper `mol-polecat-work` formula
- Monitor progress and handle coordination

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

# List tmux sessions
tmux list-sessions | grep gt-

# Check bead status
bd show <bead-id>
```

### 3. Results

Polecats follow the `mol-polecat-work` lifecycle:
1. **load-context** — Read the bead, understand the task
2. **branch-setup** — Create a working branch
3. **preflight-tests** — Verify tests pass on main
4. **implement** — Do the actual work
5. **self-review** — Review own changes
6. **run-tests** — Run tests, verify coverage
7. **cleanup-workspace** — Clean up
8. **prepare-for-review** — Prepare for merge
9. **submit-and-exit** — Push to merge queue, self-destruct

The **Refinery** agent merges polecat branches back to main. You never push directly.

## Anti-Patterns (Don't Do This)

| ❌ Don't | ✅ Do Instead |
|---|---|
| `bd create` + `gt sling` manually | Tell Mayor via `gt mayor attach` or `gt mayor mail` |
| `gt sling --hook-raw-bead` | Let Mayor apply `mol-polecat-work` formula automatically |
| Push to main directly | Let Refinery merge from the merge queue |
| Close beads manually | Polecats self-clean; Refinery closes after merge |
| Create polecats without Mayor | Mayor handles spawning and assignment |

**`--hook-raw-bead` bypasses the 9-step lifecycle.** Only use it if Mayor is actually down and you need emergency manual control.

## Quick Reference

| Action | Command |
|---|---|
| Talk to Mayor (interactive) | `gt mayor attach` |
| Message Mayor | `gt mayor mail "task description"` |
| List convoys | `gt convoy list` |
| Convoy detail | `gt convoy status <id>` |
| List agents | `gt agents list --all` |
| Peek at polecat | `tmux capture-pane -t gt-<rig>-<name> -p \| tail -30` |
| List tmux sessions | `tmux list-sessions \| grep gt-` |
| Check bead status | `bd show <bead-id>` |

## Architecture

See `references/architecture.md` for full details on Mayor, Rigs, Polecats, Hooks, Convoys, Beads, Refinery, and Witness.

## Formula Resolution (Important)

Gastown has two tools that deal with formulas differently:

- **`gt`** (orchestrator) searches 3 paths: `.beads/formulas/` (project), `~/.beads/formulas/` (user), `$GT_ROOT/.beads/formulas/` (town)
- **`bd`** (issue tracker) only searches `.beads/formulas/` relative to the current project root

When `gt sling` assigns work to a polecat, it calls `bd cook` to instantiate the `mol-polecat-work` formula. But `bd cook` runs in the rig's directory context (e.g., `~/gt/vtuber/`), so it only looks at `~/gt/vtuber/.beads/formulas/` — which doesn't exist by default for new rigs.

The formulas are installed at the town level (`~/gt/.beads/formulas/`), and `gt formula list` finds them fine. But `gt sling` doesn't pass `--search-path` to `bd cook`, so `bd` can't find them.

**The fix:** Symlink the town-level formulas into each rig during setup (included in the Setup section above). This makes the rig's `.beads/formulas/` resolve to the shared town formulas. This is the intended mechanism — rigs are designed to be self-contained, and the symlink opts them into the shared formula library.

Without this symlink, `gt sling` will log a warning like `Could not cook formula mol-polecat-work` and fall back to raw bead mode (no 9-step lifecycle).

## Troubleshooting

- **Polecat not following lifecycle**: Was it slung with `--hook-raw-bead`? That skips formula application. Re-sling through Mayor.
- **Formula not resolving (`mol-polecat-work` not found)**: See "Formula Resolution" section above. Symlink the formulas: `cd ~/gt/<rig>/.beads && ln -s ../../.beads/formulas formulas`. Verify with `cd ~/gt/<rig> && bd cook mol-polecat-work --dry-run`.
- **ICU build error on beads install**: Use `CGO_ENABLED=0 go install ...`
- **Polecat not showing in `gt agents list`**: Check tmux: `tmux list-sessions | grep gt-`
- **Need Go but no sudo**: Install to `~/local/go/` instead of `/usr/local/`
- **Polecat session frozen after work**: Claude Code sessions sometimes freeze post-completion. Check if work was committed, then kill the tmux session manually.
