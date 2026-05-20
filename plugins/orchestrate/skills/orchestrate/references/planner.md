# Planner

Operating manual for root and subplanners. Workers read their own subagent file (this plugin's `agents/orchestrate-worker.md`, loaded as the `orchestrate:orchestrate-worker` subagent type), not this.

Root and subplanners behave identically. The root reports to the user; a subplanner reports to its parent.

## Source of truth

Git and disk are the substrate.

- `<workspace>/plan.json` — task graph, config (planner-authored, never re-written by anyone else).
- `<workspace>/state.json` — task status, agent ids, branches, worktree paths, timestamps (planner-authored after each spawn / handoff).
- `<workspace>/handoffs/<task-id>.md` — worker / verifier / subplanner final message, copied verbatim from the worker's worktree.
- `<workspace>/attention.log` — operator-visible drift entries, streamed to user.

Default workspace: `.orchestrate/<rootSlug>/` at the repo root. Git-tracked.

## Phase 0 — Recovery (resume mode)

If `<workspace>/plan.json` exists when you start:

1. Load `plan.json` and `state.json`.
2. For each task with `status: running`:
   - Check if `task.worktreePath` is set in state. If so, look for `<worktreePath>/.orchestrate/handoffs/<task-id>.md`.
   - If the handoff file exists with content: adopt it (copy to workspace handoffs, parse, mark `handed-off`).
   - Otherwise: mark `status: error`, `failureMode: orphaned`, write a synthetic `<task-id>-failure.md` noting the session was interrupted.
3. Surface to the user: "Resuming `<rootSlug>`: N tasks already terminal, M still pending."
4. Proceed to Phase 2 (skip Phase 1).

If `plan.json` doesn't exist, proceed to Phase 1.

## Phase 1 — Publish tasks

Write `<workspace>/plan.json`. Schema at `schemas/plan.schema.json`. Top-level fields:

```
goal               — verbatim user goal (full agent-facing text)
rootSlug           — kebab-case identifier (e.g. "dark-mode")
baseBranch         — default startingRef for tasks without their own (typically the planner's current branch)
acceptanceCriteria — overall goal acceptance (optional)
tasks[]            — task graph
```

Per-task fields (PlanTask, three variants by `type`):

```
id             — kebab-case ASCII, unique within plan; used in handoff filename
name           — same as id (kept for parity with Cursor; can drop if redundant)
type           — worker | subplanner | verifier
scopedGoal     — outcome for this task (the only steering signal)
pathsAllowed   — globs the task may touch (optional)
pathsForbidden — globs owned by siblings (optional)
acceptance[]   — criteria the worker must hit
verify         — markdown verification recipe (optional)
startingRef    — branch/ref to start from (defaults to plan.baseBranch). For verifiers, planner auto-sets to target's branch after target hands off.
dependsOn[]    — upstream task ids; gates spawn AND triggers context relay. For verifiers, target id auto-added.
measurements[] — { name, command } pairs for quantitative drift detection (optional)
maxAttempts    — caps automatic retries (default 2)
verifies       — verifier only; target task name
```

### Planning rules

- **Merges are tasks.** Publish a worker whose `scopedGoal` says which branches to merge, conflict policy, and verification. `dependsOn` covers both branches; `startingRef` is one of them.
- **Prefer a worker unless you can name the decomposition a subplanner would do.** Subplanners exist when the slice still has internal structure or needs internal merge/verify passes.
- **One worker can carry a lot.** Workers are full Claude Code subagent sessions with their own worktree. Multi-file slices, multi-step refactors, full repro/fix/test cycles all fit in a single spawn. Default to fewer, broader workers; reach for finer granularity only when a slice is genuinely independent or has real contention risk.
- **Default to verifiers** for behavior-changing tasks. Use `type: verifier` and `verifies: <target-task-name>`. Don't manually wire `startingRef` or `dependsOn` for verifiers — planner auto-wires them after the target hands off.
- **Use `verify` for the verification recipe.** Workers read it as target behavior; verifiers inherit it from their target.
- **Add `measurements[]`** for quantitative claims. The planner reruns each command on the worker branch after handoff and logs drift.
- **Keep fan-in small.** If a task needs many upstream handoffs, publish an aggregation worker first (otherwise the downstream spawn prompt bloats).
- **Minimize path overlap.** List `pathsForbidden` when sibling ownership matters.
- **`dependsOn` is semantically meaningful.** It triggers context relay (upstream handoffs pasted into the downstream spawn prompt). Use it whenever a downstream task needs upstream findings, even if Git-level ordering doesn't strictly require it.

Then write `<workspace>/state.json` with each task `status: pending`, `attempts: 0`, `branch: null`, `worktreePath: null`.

**Surface the published task list to the user. Pause for approval.** Do not enter Phase 2 silently.

## Phase 2 — Reconcile loop

Your turn IS the heartbeat. There is no subprocess.

### Sweep step

1. **Compute ready set**: tasks where `status == pending` AND every `dependsOn` id has `status == handed-off` AND `attempts < maxAttempts`.
2. **Termination check**: if ready is empty AND no task has `status == running` (impossible mid-sweep but possible across sweeps if you parallel-spawn):
   - If any task is still `pending`, list them with their unsatisfied deps; write attention.log entries; break out of the loop.
   - Else: all terminal; break.
3. **Render spawn prompts** for each ready task (see `spawning.md` for the canonical shape). Critical step for `dependsOn`: paste each upstream handoff body verbatim under `## Upstream handoffs`.
4. **Mark tasks running** in state.json: `status: running`, increment `attempts`, set `startedAt`. Persist.
5. **Spawn ready tasks in parallel**: emit multiple `Agent` tool calls in a single assistant message. Each call:
   - `subagent_type`: `orchestrate:orchestrate-worker` | `orchestrate:orchestrate-verifier` | `orchestrate:orchestrate-planner` (the `orchestrate:` prefix is the plugin namespace; agent files inside the plugin are addressed as `<plugin-name>:<agent-name>`).
   - `isolation`: `"worktree"`
   - `description`: `"<rootSlug>/<task-name>"`
   - `prompt`: the rendered spawn prompt
6. **Process each result**:
   - Parse `worktreePath`, `worktreeBranch`, `agentId` from the result metadata (lines after the prose).
   - `Read` `<worktreePath>/.orchestrate/handoffs/<task-id>.md`.
   - If file exists: copy content to `<workspace>/handoffs/<task-id>.md`. Parse `## Status`, `## Branch`, `## Verification`. Update state.json: `status` (handed-off | partial | blocked, mapping to handoff Status), `branch`, `verification` (for verifiers, persist on target task's row), `worktreePath`, `finishedAt`.
   - If file missing: write synthetic `<task-id>-finished-no-handoff.md` (template in `handoffs.md`); set `status: error`, `failureMode: finished-no-handoff`.
   - If result indicates abnormal termination: write synthetic `<task-id>-failure.md` (template in `handoffs.md`); set `status: error`, `failureMode` from classifier.
7. **Rerun measurements** (if task declares any): for each `{name, command}`, run via Bash with `cwd = <worktreePath>`. Parse value. Compare with worker's `## Measurements` block. >10% drift or unit mismatch → append to `attention.log`.
8. **Stream attention.log deltas** to the user.
9. **Exit-on-error pause**: if any task entered `status: error` this sweep, surface to user with the failure mode and suggested action (retry / abandon / edit plan / continue with siblings). Wait for direction before next sweep.
10. Persist state.json. Loop.

### Termination

The loop exits naturally when no task is ready and none is running. Unreachable pending tasks (blocked by terminal-failed deps) stay pending; planner writes attention entries explaining the blast radius. The user can choose: fix the upstream task and re-run `/orchestrate`, or accept the partial result.

### Failure reaction

When a task transitions to `error`:

1. Read the synthetic failure handoff at `<workspace>/handoffs/<task-id>-failure.md` (or `-finished-no-handoff.md`).
2. Classify by `Failure mode`:
   - `context-limit`: retry with narrower scope (smaller `scopedGoal`, tighter `pathsAllowed`).
   - `network-drop`: retry as-is (transient).
   - `tool-error`: retry with different `model` (set `task.model` to opus/sonnet/haiku).
   - `finished-no-handoff`: read raw worker output if available; if intent is recoverable, retry with explicit reminder; else abandon.
   - `unknown`: retry as-is once; if it fails again, abandon.
3. Each retry costs another subagent run. After 2 attempts on the same task, prefer abandon (drop from plan.json, replan around) over a 3rd unless you have specific evidence the retry will succeed.
4. Update plan.json (if dropping a task or changing scope), then continue the loop.

## Phase 3 — Summary

When the loop exits:

- Counts by terminal status (`handed-off`, `error`, `cancelled`, `pending-unreachable`).
- List `handoffs/` files written.
- List branches that hold deliverable work.
- Surface any unreachable-pending warnings as actionable items.
- For root planner: print to user. For subplanner: write your own aggregated handoff (see `handoffs.md` → "Producing your own handoff (subplanner)") and exit.

Don't end your turn while the workspace has non-terminal tasks.

## Hard rules

1. **Planners do no coding.** No `Edit` or `Write` outside `<workspace>/`. No `git merge` in the planner's session. No conflict resolution. If you feel the urge, publish a worker task.
2. **Workers are isolated.** You cannot speak to them mid-run; you cannot have them speak to each other. Spawn → handoff → done.
3. **The spawn prompt is the only thing the worker sees.** No env vars to share data, no shared files outside the upstream handoff relay. Bake everything the worker needs into the prompt.
4. **`disable-model-invocation`** for the skill: load only via `/orchestrate`. Don't bootstrap from autonomous logic.
