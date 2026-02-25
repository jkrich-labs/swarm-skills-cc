---
name: swarm-planner
description: >
  [EXPLICIT INVOCATION ONLY] Creates dependency-aware implementation plans optimized for
  parallel multi-agent execution with worktree isolation. Standalone skill — invoke with /swarm-planner.
metadata:
  invocation: explicit-only
---

# Swarm-Ready Planner

Create implementation plans with explicit task dependencies optimized for parallel agent execution in Claude Code. Plans produced by this skill are executed by the `swarm-executor` skill.

**Announce at start:** "I'm using the swarm-planner skill to create a dependency-aware parallel implementation plan."

## CRITICAL CONSTRAINTS

**You MUST NOT call `EnterPlanMode` or `ExitPlanMode` during this skill.** This skill operates in normal mode and manages its own flow. Plan mode restricts Write/Edit tools and has no clean exit.

## Core Principles

1. **Explore Codebase**: Investigate architecture, patterns, existing implementations, dependencies, and frameworks in use
2. **Fresh Documentation First**: Use Context7 for ANY external library, framework, or API before planning tasks
3. **Ask Questions**: Clarify ambiguities — seek clarification on scope, constraints, or priorities at any point
4. **Explicit Dependencies**: Every task declares what it depends on, enabling maximum parallelization
5. **Atomic Tasks**: Each task is independently executable by a single agent in an isolated worktree
6. **File Isolation**: Tasks in the same wave MUST NOT touch the same files — this is a hard requirement for worktree-based parallel execution
7. **Review Before Yield**: A subagent reviews the plan for gaps before finalizing

## Process

### 1. Research

**Codebase investigation:**
- Architecture, patterns, existing implementations
- Dependencies and frameworks in use
- File ownership boundaries (which files belong to which subsystem)

Use the `Explore` subagent type for broad codebase exploration, or `Glob`/`Grep` for targeted searches.

### 1a. Optional: Stop for Clarification

- If the architecture is unclear or missing, STOP and ask the user for input before moving on. Always offer recommendations with your questions.
- If architecture is clear, skip to step 2.

### 2. Documentation

**Documentation retrieval (REQUIRED for external dependencies):**

Use Context7 MCP to fetch current docs for any libraries/frameworks/APIs that are or will be used in the project. This ensures version-accurate APIs, correct parameters, and current best practices.

### 3. STOP and Request User Input

When anything is unclear or could reasonably be done multiple ways:
- Stop and ask clarifying questions immediately
- Do not make assumptions about scope, constraints, or priorities
- Questions should reduce risk and eliminate ambiguity
- Always offer recommendations with your questions
- One question per message — do not overwhelm

### 4. Create Dependency-Aware Plan

Structure the plan with explicit task dependencies. Critical rules:

**Dependency Rules:**
- Every task MUST have a `depends_on` field (empty `[]` for root tasks)
- Tasks with empty/satisfied dependencies can run in parallel
- Tasks in the same wave MUST operate on different files

**File Isolation Rules (CRITICAL for worktree execution):**
- Map every task to its exact file paths
- If two tasks need to modify the same file, one MUST depend on the other
- Shared files = forced serialization. Design tasks to minimize shared files.
- When unavoidable, document WHY the dependency exists

### 5. Save Plan

Save to `docs/plans/YYYY-MM-DD-<topic>-swarm.md` in the project root.

Create the `docs/plans/` directory if it doesn't exist.

### 6. Save Task Persistence File

Save to `docs/plans/YYYY-MM-DD-<topic>-swarm.md.tasks.json` alongside the plan:

```json
{
  "planPath": "docs/plans/YYYY-MM-DD-<topic>-swarm.md",
  "tasks": [
    {"id": "T1", "subject": "Task title", "status": "pending", "dependsOn": []},
    {"id": "T2", "subject": "Task title", "status": "pending", "dependsOn": ["T1"]}
  ],
  "waves": [
    {"wave": 1, "tasks": ["T1", "T3"]},
    {"wave": 2, "tasks": ["T2", "T4"]}
  ],
  "lastUpdated": "<ISO timestamp>"
}
```

### 7. Create Native Tasks

**REQUIRED FIRST:** Load the task tools before using them:
```
ToolSearch: "select:TaskCreate"
ToolSearch: "select:TaskUpdate"
ToolSearch: "select:TaskList"
```

For each task in the plan, create a native task. **Track the mapping between plan IDs (T1, T2...) and the native task IDs returned by TaskCreate** — you need this mapping for setting dependencies.

```
TaskCreate:
  subject: "T1: [Task Name]"
  description: |
    [Full task content — files, description, validation, acceptance criteria]
  activeForm: "Implementing [Task Name]"
# Returns a native task ID — record it: T1 → native-id-1
```

After all tasks created, set dependency relationships using the **native task IDs** (not plan IDs):

```
TaskUpdate:
  taskId: [native-id-for-dependent-task]
  addBlockedBy: [native-ids-for-prerequisite-tasks]
```

Call `TaskList` to verify the dependency structure looks correct.

### 8. Subagent Review

Spawn a subagent to review the plan:

```
Task(subagent_type="general-purpose", prompt="""
Review this implementation plan for:
1. Missing dependencies between tasks
2. File isolation violations — tasks in the same wave editing the same files
3. Ordering issues that would cause failures
4. Missing error handling or edge cases
5. Gaps, holes, gotchas

CRITICAL: Check that no two tasks in the same parallel wave touch the same files.
Tasks will run in isolated git worktrees — file conflicts between parallel tasks
will cause merge failures.

Provide specific, actionable feedback. Do not ask questions.

Plan location: [file path]
Context: [brief context about the task]
""")
```

If the subagent provides actionable feedback:
1. Revise the plan file
2. Regenerate the `.tasks.json` persistence file to match
3. Update native tasks: use `TaskUpdate` to modify descriptions/subjects for existing tasks. If tasks were added or removed, delete stale tasks and create new ones as needed.
4. Re-verify with `TaskList` before yielding

### 9. Yield to User

Present the completed plan and tell the user:

```
Plan saved to docs/plans/YYYY-MM-DD-<topic>-swarm.md

To execute: /swarm-executor docs/plans/YYYY-MM-DD-<topic>-swarm.md
To execute a subset: /swarm-executor docs/plans/YYYY-MM-DD-<topic>-swarm.md T1 T3 T5
```

## Plan Template

````markdown
# Swarm Plan: [Task Name]

> **For Claude:** REQUIRED SUB-SKILL: Use swarm-executor to implement this plan.

**Generated**: [Date]
**Goal**: [One sentence describing what this builds]
**Architecture**: [2-3 sentences about approach]
**Tech Stack**: [Key technologies/libraries]

---

## Overview
[Summary of task and approach]

## Prerequisites
- [Tools, libraries, access needed]

## Dependency Graph

```
T1 ──┬── T3 ──┐
     │        ├── T5 ── T6 ── T7
T2 ──┴── T4 ──┘
```

## Tasks

### T1: [Name]
- **depends_on**: []
- **files**: [exact file paths — create/modify/test]
- **description**: [what to do, with enough context for an agent with zero codebase knowledge]
- **acceptance_criteria**:
  - [Specific, testable criterion]
  - [Specific, testable criterion]
- **validation**: [exact commands to verify completion]
- **status**: pending
- **log**: []
- **files_edited**: []

### T2: [Name]
- **depends_on**: []
- **files**: [exact file paths]
- **description**: [what to do]
- **acceptance_criteria**:
  - [Criterion]
- **validation**: [how to verify]
- **status**: pending
- **log**: []
- **files_edited**: []

### T3: [Name]
- **depends_on**: [T1]
- **files**: [exact file paths]
- **description**: [what to do]
- **acceptance_criteria**:
  - [Criterion]
- **validation**: [how to verify]
- **status**: pending
- **log**: []
- **files_edited**: []

[... continue for all tasks ...]

## Parallel Execution Waves

| Wave | Tasks | Can Start When | Files Touched |
|------|-------|----------------|---------------|
| 1 | T1, T2 | Immediately | T1: src/a.py, T2: src/b.py |
| 2 | T3, T4 | Wave 1 complete | T3: src/c.py, T4: src/d.py |
| 3 | T5 | T3, T4 complete | src/e.py |

**File isolation verified**: No two tasks in the same wave touch the same files.

## Testing Strategy
- [How to test]
- [What to verify]
- [Full test suite command]

## Risks & Mitigations
- [What could go wrong + how to handle]
````

## Important Reminders

- Every task MUST have explicit `depends_on` field
- Every task MUST list exact file paths
- Tasks in the same wave MUST NOT share files
- Root tasks (no dependencies) execute in parallel immediately
- Do NOT implement — only create the plan
- Always use Context7 for external dependencies before finalizing tasks
- Always ask questions where ambiguity exists
- The plan must contain enough context for an agent with zero prior knowledge to implement each task
