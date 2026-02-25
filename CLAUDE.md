# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **Claude Code plugin** that provides two skills for dependency-aware parallel planning and execution. There is no buildable code, tests, or compilation — the entire plugin consists of Markdown skill definitions and JSON plugin metadata.

The plugin is distributed via the Claude Code marketplace as `swarm-skills@swarm-skills-cc`.

## Repository Structure

- `.claude-plugin/plugin.json` — Plugin identity, version, and metadata
- `.claude-plugin/marketplace.json` — Marketplace listing configuration
- `skills/swarm-planner/SKILL.md` — Planner skill: creates dependency-aware plans with wave tables and file isolation verification
- `skills/swarm-executor/SKILL.md` — Executor skill: orchestrates parallel execution using worktree-isolated subagents

## How the Skills Work Together

1. **swarm-planner** produces a plan at `docs/plans/YYYY-MM-DD-<topic>-swarm.md` with a companion `.tasks.json` state file. Plans contain tasks with explicit `depends_on` arrays, file ownership mappings, and parallel execution wave tables.

2. **swarm-executor** reads a plan, identifies unblocked tasks, and dispatches them as parallel `Task` subagents with `isolation: "worktree"`. After each wave completes, it merges worktree branches, validates results, updates state, and launches the next wave.

Key invariants enforced across both skills:
- Tasks in the same wave must not touch the same files (file isolation) — the plan file is the known exception, handled by Tier 1 conflict resolution
- Merge conflicts are auto-resolved using tiered strategies (plan file → additive → subagent resolver → user escalation)
- Worktrees are cleaned up immediately after each wave — none survive past their wave
- Neither skill uses `EnterPlanMode`/`ExitPlanMode`
- Subagents never push — only commit

## Editing Skills

Skill files use YAML frontmatter (`name`, `description`, `disable-model-invocation`) followed by Markdown. The frontmatter fields are functional — `disable-model-invocation: true` prevents the skill from being auto-triggered by the model.

When modifying skills, preserve the plan template structure in swarm-planner and the subagent prompt template in swarm-executor — these are the contracts between the two skills.

## Installation for Local Development

```bash
claude --plugin-dir ./path/to/swarm-skills-cc
```
