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

1. `/swarm-skills:swarm-planner` — explores the codebase, asks clarifying questions, produces a dependency-aware plan with wave tables and file isolation verification
2. `/swarm-skills:swarm-executor <plan-path>` — executes the plan in parallel waves using worktree-isolated subagents, auto-resolves merge conflicts, and tracks progress via native tasks

## Key Features

- **Dependency-aware plans** with `depends_on` arrays and wave tables
- **File isolation verification** — tasks in the same wave cannot touch the same files
- **Worktree isolation** — each parallel task runs in its own git worktree
- **Tiered merge conflict resolution** — plan file conflicts, additive conflicts, and code conflicts are auto-resolved in escalating tiers; user escalation is the last resort
- **Resume capability** — interrupted executions pick up from `.tasks.json` state and `in_progress` tracking
- **Plan-scoped native tasks** — tasks are prefixed with a plan slug to prevent cross-plan collisions on resume
- **Strict worktree cleanup** — no worktree survives past its wave

## License

MIT
