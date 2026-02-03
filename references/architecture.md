# Gastown Architecture

## Components

### Mayor 🎩
Primary AI coordinator — **your main interface to Gastown**. A Claude Code instance with full context about workspace, projects, and agents. Handles:
- Breaking tasks into beads
- Creating convoys for tracking
- Slinging beads to polecats with proper formulas
- Monitoring progress and handling coordination
- Rig bootstrapping and formula resolution

Start interactive session with `gt mayor attach`. Send async messages with `gt mayor mail "..."`.

### Town 🏘️
Workspace directory (`~/gt/`). Contains all projects, agents, and configuration.

### Rigs 🏗️
Project containers. Each rig wraps a git repository and manages its associated agents. Add with `gt rig add <name> <repo>`.

### Polecats 🦨
Ephemeral worker agents. Spawned by Mayor via `gt sling`, they follow the `mol-polecat-work` lifecycle:
1. Load context and verify assignment
2. Set up working branch
3. Verify preflight tests pass
4. Implement the solution
5. Self-review changes
6. Run tests and verify coverage
7. Clean up workspace
8. Prepare work for review
9. Submit to merge queue and self-destruct

Each gets:
- Own git branch (`polecat/<name>/<bead>@<hash>`)
- Own tmux session (`gt-<rig>-<name>`)
- Access to the rig's codebase
- Mail-based communication with other agents

**Self-cleaning:** Polecats push work, submit to MQ, nuke themselves, and exit. No idle state.

### Formulas 📜
Workflow templates that define step-by-step lifecycles. Key formulas:
- `mol-polecat-work` — Standard polecat work lifecycle (9 steps)
- `shiny` — "Engineer in a Box" design-first workflow
- `mol-polecat-code-review` — Code review workflow
- `mol-witness-patrol` — Witness monitoring loop
- `mol-refinery-patrol` — Merge queue processing loop

Formulas are applied automatically when slinging through Mayor. Bypassing them (via `--hook-raw-bead`) skips the structured lifecycle.

### Hooks 🪝
Git worktree-based persistent storage. Work state survives crashes and restarts. When a bead is slung, it's "hooked" — attached to a polecat's worktree.

### Convoys 🚚
Work tracking bundles. Group multiple beads for coordinated delivery. Auto-close when all tracked beads complete. Create with `gt convoy create`.

### Beads 📿
Git-backed issue tracking. Bead IDs use prefix + 5-char alphanumeric (e.g., `vt-abc12`). The prefix indicates the rig. Create with `bd create`.

### Refinery 🏭
Merge queue processor. Handles merging polecat branches back to main. Runs as a persistent agent. You never push to main directly — Refinery handles it.

### Witness 🦉
Monitoring agent. Watches polecat lifecycles, catches stuck/crashed workers, and reports issues to Mayor.

## Data Flow

```
You → Mayor → creates Convoy with Beads
                  → slings Beads to Polecats (with mol-polecat-work formula)
                      → Polecats work on branches (9-step lifecycle)
                      → Polecats commit + submit to merge queue + self-destruct
                  → Refinery merges branches to main
              → Convoy auto-closes when all beads done
          → Mayor reports results
```

## Scaling

Gastown comfortably scales to 20-30 concurrent agents. Each polecat is an independent Claude Code process with its own context, so they don't interfere with each other. The git-backed state means work persists even if agents crash.

## Common Mistakes

1. **Bypassing Mayor** — Manually creating beads and slinging them skips formula application, convoy tracking, and coordination. Always go through Mayor.
2. **Using `--hook-raw-bead`** — This skips the `mol-polecat-work` lifecycle. Polecats won't follow the 9-step process and may not self-clean properly.
3. **Pushing to main** — Only Refinery pushes to main via the merge queue. Polecats work on their own branches.
4. **Closing beads manually** — Refinery closes beads after successful merge.
