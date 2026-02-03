# Gastown Architecture

## Components

### Mayor 🎩
Primary AI coordinator. A Claude Code instance with full context about workspace, projects, and agents. Start interactive session with `gt mayor attach`.

### Town 🏘️
Workspace directory (`~/gt/`). Contains all projects, agents, and configuration.

### Rigs 🏗️
Project containers. Each rig wraps a git repository and manages its associated agents. Add with `gt rig add <name> <repo>`.

### Polecats 🦨
Ephemeral worker agents. Spawned via `gt sling`, they complete a task and disappear. Each gets:
- Own git branch (`polecat/<name>/<bead>@<hash>`)
- Own tmux session (`gt-<rig>-<name>`)
- Access to the rig's codebase
- Mail-based communication with other agents

### Hooks 🪝
Git worktree-based persistent storage. Work state survives crashes and restarts. When a bead is slung, it's "hooked" — attached to a polecat's worktree.

### Convoys 🚚
Work tracking bundles. Group multiple beads for coordinated delivery. Auto-close when all tracked beads complete. Create with `gt convoy create`.

### Beads 📿
Git-backed issue tracking. Bead IDs use prefix + 5-char alphanumeric (e.g., `vt-abc12`). The prefix indicates the rig. Create with `bd create`.

### Refinery 🏭
Merge queue processor. Handles merging polecat branches back to main. Runs as a persistent agent.

### Witness 🦉
Monitoring agent. Watches polecat lifecycles and reports issues.

## Data Flow

```
You → Mayor → creates Convoy with Beads
                  → slings Beads to Polecats
                      → Polecats work on branches
                      → Polecats commit + report
                  → Refinery merges branches
              → Convoy auto-closes
          → Mayor reports results
```

## Scaling

Gastown comfortably scales to 20-30 concurrent agents. Each polecat is an independent Claude Code process with its own context, so they don't interfere with each other. The git-backed state means work persists even if agents crash.
