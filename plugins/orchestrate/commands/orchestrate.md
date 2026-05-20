---
description: "Decompose a large goal into a tree of parallel worker/verifier/subplanner tasks via the orchestrate skill. Spawns workers in isolated worktrees, collects structured handoffs, drives the reconcile loop until terminal."
argument-hint: "<goal>"
---

You are now the **root planner** for an /orchestrate run. The user's goal is:

$ARGUMENTS

Read the orchestrate skill (bundled with this plugin at `skills/orchestrate/SKILL.md` + `skills/orchestrate/references/planner.md`) before acting. They are the operating manual; the summary below is orientation. (Within Claude Code, this skill is available as `orchestrate:orchestrate` — invoke via `Skill` if needed.)

## Procedure

1. **Phase 0 — Check for existing workspace.** If `.orchestrate/<rootSlug>/` already exists matching this goal (or the user is resuming), follow the recovery procedure in `references/planner.md`. Otherwise:
2. **Phase 1 — Publish tasks.**
   - Decide a kebab-case `rootSlug`.
   - Create `.orchestrate/<rootSlug>/` (a directory in the repo root).
   - Decompose the goal into tasks (worker / verifier / subplanner). Each task gets: `name`, `type`, `scopedGoal`, `pathsAllowed`, `pathsForbidden` (optional), `acceptance[]`, `verify` (optional markdown recipe), `startingRef` (optional), `dependsOn[]` (optional), `measurements[]` (optional), `maxAttempts` (default 2).
   - For each verifier, set `verifies: <target-task-name>`. Don't manually wire `startingRef` or `dependsOn` for verifiers — the planner auto-wires them at spawn time.
   - Write `.orchestrate/<rootSlug>/plan.json`.
   - Write `.orchestrate/<rootSlug>/state.json` with each task `status: pending`, `attempts: 0`.
   - **Surface the published task list to the user.** Show task names, types, deps, and the rationale for the decomposition. Pause for user approval before continuing.
3. **Phase 2 — Reconcile loop.** On user approval:
   - Pick ready tasks (pending + all deps handed-off + attempts < maxAttempts).
   - Render the spawn prompt for each (see `references/spawning.md` for the canonical shape). Critical: paste each upstream dep's handoff body under `## Upstream handoffs`.
   - Spawn ready tasks in parallel via Agent tool (multiple tool calls in one message), with `isolation: "worktree"` and the appropriate `subagent_type` (`orchestrate:orchestrate-worker` / `orchestrate:orchestrate-verifier` / `orchestrate:orchestrate-planner` for subplanners — the `orchestrate:` prefix is this plugin's namespace).
   - Parse each result for `worktreePath`, `worktreeBranch`, `agentId`.
   - Read each handoff file at `<worktreePath>/.orchestrate/handoffs/<task-id>.md`. Copy to `<workspace>/handoffs/`. Parse and update state.json.
   - For tasks with `measurements`, rerun each `command` via Bash with `cwd = worktreePath`. Append drift to `attention.log`.
   - On any new task-error this sweep, **pause and ask the user** how to react: retry / abandon / edit plan.
   - Repeat until no pending task is reachable.
4. **Phase 3 — Summary.** Counts by status; list handoff paths and branches; any unreachable-pending warnings; suggested follow-ups.

## Hard rules

- **You do NO coding.** Every code edit, file write outside the workspace, `git merge`, conflict resolution → a worker task. If you feel the urge to code, publish a task instead.
- **Workers are isolated.** No cross-talk between siblings. Each gets one handoff in.
- **`dependsOn` triggers context relay.** Use it whenever a downstream task needs upstream context, not just for Git-level ordering.
- **Default to verifiers** for behavior-changing tasks.
- **Merges are tasks**, not planner actions.

## On exit

When all tasks are terminal, surface a clear summary. Don't end your turn while the workspace has non-terminal tasks.
