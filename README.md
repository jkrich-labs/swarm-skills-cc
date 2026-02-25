# swarm-skills

Dependency-aware parallel planning and execution plugin for Claude Code.

## Install

### From marketplace (recommended)

```bash
# Add the marketplace
/plugin marketplace add jkrich-labs/swarm-skills-cc

# Install the plugin
/plugin install swarm-skills@swarm-skills-cc
```

### Local development

```bash
claude --plugin-dir ./path/to/swarm-skills-cc
```

## Skills

### swarm-planner

Creates implementation plans with explicit task dependencies optimized for parallel multi-agent execution. Plans include dependency graphs, parallel execution wave tables, and file isolation verification.

```
/swarm-skills:swarm-planner
```

### swarm-executor

Orchestrates parallel execution of swarm plans. Launches subagents in dependency-ordered waves with git worktree isolation, validates after each wave, and cleans up worktrees immediately.

```
/swarm-skills:swarm-executor docs/plans/2026-02-25-feature-swarm.md
/swarm-skills:swarm-executor docs/plans/2026-02-25-feature-swarm.md T1 T3 T5
```

## Workflow

1. `/swarm-skills:swarm-planner` — research, ask questions, produce dependency-aware plan
2. `/swarm-skills:swarm-executor <plan-path>` — execute plan in parallel waves with worktree isolation

## Key Features

- **Dependency-aware plans** with `depends_on` arrays and wave tables
- **File isolation verification** — tasks in the same wave cannot touch the same files
- **Worktree isolation** — each parallel task runs in its own git worktree
- **Strict worktree cleanup** — no worktree survives past its wave
- **Resume capability** — interrupted executions pick up from `.tasks.json`
- **Native task integration** — `TaskCreate`/`TaskUpdate` for CLI-visible progress
- **Merge conflict escalation** — never auto-resolves, always asks the user

## License

MIT
