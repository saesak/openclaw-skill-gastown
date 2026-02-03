# 🏭 Gastown Skill for OpenClaw

An [OpenClaw](https://github.com/openclaw/openclaw) skill that integrates [Gas Town](https://github.com/steveyegge/gastown) — Steve Yegge's multi-agent coding orchestrator — as the default tool for non-trivial coding tasks.

## What it does

Instead of your AI agent one-shotting complex coding tasks (and often getting them wrong), this skill teaches it to delegate to Gastown, which coordinates **multiple Claude Code agents working in parallel** with:

- 🔀 **Parallel execution** — 20-30 agents working simultaneously
- 💾 **Persistent state** — git-backed hooks survive crashes
- 📋 **Work tracking** — beads + convoys for organized task management
- 🔄 **Iteration** — agents can test, fix errors, and iterate (not just one-shot)
- 🤝 **Coordination** — agents communicate via mailboxes

## Install

### From ClawHub (recommended)

```bash
clawhub install gastown
```

### Manual

Clone this repo into your OpenClaw skills directory:

```bash
git clone https://github.com/YOUR_USERNAME/openclaw-skill-gastown.git \
  ~/.openclaw/skills/gastown
```

## Prerequisites

- [Claude Code CLI](https://claude.ai/code)
- tmux 3.0+
- Go 1.23+ (auto-installed by setup script)

## Setup

The skill includes a setup script that installs all dependencies:

```bash
# From the skill directory
./scripts/setup.sh
```

## Usage

Once installed, your OpenClaw agent will automatically use Gastown for coding tasks. It will:

1. Create beads (work items) for each task
2. Bundle them into convoys
3. Sling them to polecat agents (Claude Code instances)
4. Monitor progress and report results

## Credits

- [Gas Town](https://github.com/steveyegge/gastown) by Steve Yegge
- [OpenClaw](https://github.com/openclaw/openclaw)
- [ClawHub](https://clawhub.com)

## License

MIT
