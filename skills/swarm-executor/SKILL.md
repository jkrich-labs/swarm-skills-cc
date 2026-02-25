---
name: swarm-executor
description: >
  Orchestrates parallel execution of swarm plans using dependency-ordered waves with
  worktree-isolated subagents. Invoke with /swarm-skills:swarm-executor <plan-path> [task-ids...].
disable-model-invocation: true
---

# Swarm Executor

Orchestrator for parallel plan execution. Parses swarm plan files, launches subagents in dependency-ordered waves using worktree isolation, validates after each wave, and repeats until all tasks complete.

**Announce at start:** "I'm using the swarm-skills:swarm-executor skill to execute this swarm plan in parallel waves."

## CRITICAL CONSTRAINTS

**You MUST NOT call `EnterPlanMode` or `ExitPlanMode` during this skill.** This skill operates in normal mode and needs Write/Edit access for plan updates and merge operations.

**Worktree cleanup is NON-NEGOTIABLE.** Every worktree created by this skill MUST be deleted before the wave is considered complete. No worktree survives past its wave.

## Process

### Step 1: Parse Request

Extract from user input:
1. **Plan file path**: The markdown swarm plan to read
2. **Task subset** (optional): Specific task IDs to run (e.g., `T1 T3 T5`)

If no subset provided, run the full plan.

### Step 1b: Load Task Tools

**REQUIRED:** Load task and worktree tools before proceeding:
```
ToolSearch: "select:TaskCreate"
ToolSearch: "select:TaskUpdate"
ToolSearch: "select:TaskList"
```

### Step 2: Load Plan and Restore State

1. Read the plan file fully
2. Look for `.tasks.json` persistence file at `<plan-path>.tasks.json`
3. Call `TaskList` to check for existing native tasks
4. **Resume logic — match native tasks to this plan** by checking if task subjects contain the plan's task IDs (e.g., "T1:", "T2:"):
   - If `.tasks.json` exists AND no matching native tasks found: recreate native tasks from JSON using `TaskCreate`, restore `blockedBy` with `TaskUpdate`
   - If matching native tasks exist: verify they match the plan, resume from first `pending`/`in_progress` task
   - If neither: bootstrap from plan (Step 2b)
5. If a task subset was requested, filter to only those IDs and their required dependencies
6. Review the plan critically — raise concerns with the user before starting

### Step 2b: Bootstrap Tasks from Plan

If no persistence state exists:

1. Parse the plan for `### T<id>:` task headers
2. For each task, extract: ID, name, `depends_on`, files, description, acceptance criteria, validation
3. Create native tasks via `TaskCreate` for each
4. Set `blockedBy` relationships via `TaskUpdate`
5. Save initial `.tasks.json` persistence file
6. Call `TaskList` to verify dependency structure

### Step 3: Identify Current Wave

1. Collect all tasks with status `pending`
2. A task is **unblocked** if every ID in its `depends_on` list has status `completed`
3. All currently unblocked tasks form the current wave
4. **Pre-flight check**: Verify no two tasks in this wave touch the same files. If they do, STOP and alert the user — the plan has a file isolation violation.

### Step 4: Launch Wave

For each unblocked task in the current wave, dispatch a subagent using the `Task` tool:

- **subagent_type**: `general-purpose`
- **isolation**: `worktree`
- **All unblocked tasks in a single message** — this is how Claude Code achieves true parallelism

**CRITICAL: Send ALL Task tool calls for the wave in ONE message.** Do not send them sequentially.

#### Subagent Prompt Template

```
You are implementing a specific task from a swarm development plan.
You are working in an isolated git worktree — your changes will be merged by the orchestrator.

## Context
- Plan: [filename]
- Goal: [relevant overview from plan]
- Your task's dependencies (already completed): [list completed prerequisite tasks and their summaries]
- Tasks that depend on yours: [downstream task IDs, for awareness]
- Risks: [relevant risks from plan]

## Your Task
**Task [ID]: [Name]**

Files: [exact file paths from plan]
Description: [full description from plan]

Acceptance Criteria:
[list from plan]

Validation:
[commands/tests from plan]

## Instructions
1. Read the plan file and understand the full context
2. Read all relevant files before making changes
3. Implement changes for all acceptance criteria
4. Keep work atomic and committable
5. For each file: read first, edit carefully, preserve formatting
6. Run validation commands if specified
7. **Update the plan file** — mark your task's status as `completed` and fill in:
   - `log`: concise summary of what you did
   - `files_edited`: list of files you modified/created
8. Commit your work with a descriptive message
   - Stage ONLY the files you changed plus the plan file — never use `git add -A` or `git add .`
   - NEVER push. ONLY commit.
9. **Double-check** that the plan file is updated and all your work is committed before returning
10. Return a summary of:
    - Files modified/created
    - Changes made
    - How each acceptance criterion is satisfied
    - Validation output (or note if deferred)
    - Any blockers or concerns for downstream tasks

## Important
- Be careful with file paths
- Stop and describe blockers — do not guess or hack around them
- Focus exclusively on YOUR task — do not touch files outside your scope
- You are in a worktree — other agents are working in parallel on other tasks
```

### Step 5: Collect Results and Merge

As each subagent returns:

1. **Read the result** — check for success, blockers, or failures
2. **If the subagent made changes:** The Task tool returns `worktree_path` and `worktree_branch` in its result when `isolation: "worktree"` was used and changes were committed.
   a. Merge the worktree branch into the working branch:
      ```bash
      git merge <worktree-branch> --no-ff -m "swarm: merge T<id> - <task name>"
      ```
   b. **If merge conflict:** follow the **Tiered Conflict Resolution** process below.
   c. After successful merge (or successful conflict resolution), delete the worktree and branch immediately:
      ```bash
      git worktree remove <worktree-path> --force
      git branch -D <worktree-branch>
      ```
3. **If the subagent made no changes** (worktree auto-cleaned by Task tool): note the failure or no-op
4. **If the subagent reported a blocker**: mark task as blocked, log the reason

**CLEANUP GUARANTEE:** After processing each subagent result, verify the worktree is gone:
```bash
git worktree list
```
If any stale worktrees remain from this wave, remove them immediately.

#### Tiered Conflict Resolution

When a merge produces conflicts, resolve them using escalating tiers. Log which tier was used for each file in the wave summary.

**Tier 1 — Plan file conflicts:**
If the only conflicting file is the plan file (`docs/plans/*.md`):
1. Accept the incoming version: `git checkout --theirs <plan-file>`
2. Re-apply the already-merged task's status updates using Edit (each task edits its own `### T<id>:` section, so these are non-overlapping)
3. `git add <plan-file>` and continue the merge

**Tier 2 — Additive conflicts:**
If a conflict region consists entirely of additions from both sides (no lines deleted or modified relative to the base), combine both additions:
1. Get the base version: `git show :1:<file> > /tmp/base`
2. Check that both sides only add lines relative to base (no deletions or modifications)
3. If purely additive: combine both sides' added lines, replace the conflict region, `git add <file>`
4. If not purely additive: fall through to Tier 3

This covers import blocks, barrel exports (`export * from`), and config arrays where both agents appended entries.

**Tier 3 — Subagent resolver:**
For code conflicts that aren't purely additive, spawn a resolver subagent:

```
Task(subagent_type="general-purpose", prompt="""
You are resolving a merge conflict between two parallel tasks from a swarm plan.

## Conflict
File: <path>
Base version (before either task):
<output of: git show :1:<file>>

Ours (already merged — task <ID>):
<output of: git show :2:<file>>

Theirs (incoming — task <ID>):
<output of: git show :3:<file>>

## Task Context
**Task <ID> (ours):** <description + acceptance criteria from plan>
**Task <ID> (theirs):** <description + acceptance criteria from plan>

## Instructions
1. Understand both tasks' intent from the context above
2. Produce the complete merged file that preserves BOTH tasks' changes
3. Do not drop functionality from either side
4. If the changes are truly incompatible (same logic rewritten differently),
   respond with only: CANNOT_RESOLVE: <reason>
5. Otherwise, write the resolved file using the Write tool — nothing else
""")
```

After the resolver returns:
1. If CANNOT_RESOLVE: fall through to Tier 4
2. Stage the resolved file: `git add <file>`
3. Run validation commands from **both** conflicting tasks (if specified in the plan)
4. If validation passes: accept the resolution, continue the merge
5. If validation fails: restore conflict state with `git checkout --merge <file>`, fall through to Tier 4

**Tier 4 — User escalation:**
If all automated tiers fail:
1. STOP and show the user: which file, which tasks, what was attempted, and why auto-resolution failed
2. Let the user resolve manually
3. Resume with `/swarm-skills:swarm-executor <plan-path>` (picks up from `.tasks.json`)

### Step 6: Validate and Update State

After ALL subagents in the wave have returned and been merged:

1. **Validate each task's output** against its acceptance criteria
2. **Verify plan file updates** — subagents are instructed to update their own task status in the plan file. After merging, verify each completed task has:
   - `status`: `completed`
   - `log`: summary of what was done
   - `files_edited`: actual files modified
   If a subagent failed to update the plan, fill in the missing fields from the subagent's return summary.
3. **Update `.tasks.json`** — set status to `completed` for finished tasks, update `lastUpdated`
4. **Update native tasks** — `TaskUpdate` with `status: completed` for each
5. **If a task failed or was blocked:**
   - Mark it `blocked` or `failed` in all three places (plan, JSON, native task)
   - Log the reason
   - Determine if downstream tasks are now blocked
6. **Commit the plan file updates:**
   ```bash
   git add docs/plans/<plan-file> docs/plans/<plan-file>.tasks.json
   git commit -m "swarm: wave N complete — T<ids> done"
   ```

### Step 7: Repeat or Complete

1. Re-read the plan to find newly unblocked tasks
2. If unblocked tasks exist: return to Step 3
3. If all tasks complete: proceed to Step 8
4. If tasks remain but none are unblocked (all blocked/failed): STOP and report to user

### Step 8: Execution Summary

Present the final summary:

````markdown
# Swarm Execution Summary

## Plan: [plan name]
## Tasks Assigned: [N]
## Tasks Completed: [N]
## Waves Executed: [N]

### Overall Status
[One sentence: all complete, partially complete with blockers, etc.]

### Completed Tasks
| Task | Name | Files Modified | Summary |
|------|------|----------------|---------|
| T1 | [name] | [files] | [brief summary] |

### Failed/Blocked Tasks
| Task | Name | Issue | Resolution Needed |
|------|------|-------|-------------------|
| T5 | [name] | [what went wrong] | [what needs to happen] |

### Wave Execution Log
- **Wave 1** (T1, T2): [outcome]
- **Wave 2** (T3, T4): [outcome]

## Files Modified (All Tasks)
[complete list]

## Worktree Cleanup
All worktrees created during execution have been removed. ✓

## Next Steps
[recommendations — run full test suite, manual review areas, etc.]
````

After presenting the summary, call `TaskList` to display the final native task state.

## Error Handling

### Merge Conflicts
- Conflicts are resolved automatically using the **Tiered Conflict Resolution** process (see Step 5)
- **Tier 1** (plan file) and **Tier 2** (additive) resolve without subagents
- **Tier 3** spawns a resolver subagent for code conflicts, validated against both tasks' acceptance criteria
- **Tier 4** escalates to the user only when automated resolution fails validation
- On user escalation, resume with `/swarm-skills:swarm-executor <plan-path>` (picks up from `.tasks.json`)

### Subagent Failure
- Mark task as `failed` in plan, JSON, and native task
- Check if downstream tasks are now blocked (mark them `blocked` too)
- Report to user with the failure details
- Other tasks in the current wave are already running (they were dispatched in parallel) — let them complete, then assess before launching the next wave

### Parse Failure
- Show what was attempted and what failed
- List available task IDs from the plan
- Ask user for clarification

### Stale Worktrees
- At the START of every execution, run `git worktree list` and check for stale worktrees from previous runs
- If found: alert the user and offer to clean them up before proceeding

## Resume Capability

This skill supports resuming interrupted executions:

1. Re-invoke with the same plan path: `/swarm-skills:swarm-executor <plan-path>`
2. The skill reads `.tasks.json` and native tasks to determine what's already done
3. Picks up from the first pending wave
4. No work is duplicated

## Example Usage

```
/swarm-skills:swarm-executor docs/plans/2026-02-25-auth-system-swarm.md
/swarm-skills:swarm-executor docs/plans/2026-02-25-auth-system-swarm.md T3 T5 T7
```

## Important Reminders

- ALL wave tasks dispatched in ONE message — this is how parallelism works
- Worktree isolation is mandatory — every task gets `isolation: "worktree"`
- Worktree cleanup is immediate — no worktree survives past its wave
- Merge conflicts are auto-resolved in tiers — user escalation is the last resort, not the first
- Plan file is the source of truth — always update it after each wave
- `.tasks.json` enables resume — always keep it in sync
- Never push — only commit. User decides when to push.
