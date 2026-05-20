# Handoffs

Handoffs are the only way information moves between nodes. Workers produce one; planners read them and decide. No shared branch, no status API, no cross-sibling chatter. That uniformity is what keeps the tree in motion without global coordination.

## File location

Workers write their handoff to `.orchestrate/handoffs/<task-id>.md` inside their worktree (NOT inside `<workspace>/handoffs/` — that path doesn't exist in the worker's worktree because the central workspace is in the planner's worktree). The worker commits this file on its branch.

The planner, after `claude-bg-wait` returns, reads the file from `<worktreePath>/.orchestrate/handoffs/<task-id>.md` and copies it to the central `<workspace>/handoffs/<task-id>.md`.

This double-storage is intentional:
- Worker-side copy: preserves handoff in the worker's branch history for forensics; survives if the planner crashes between read and copy.
- Central copy: easy operator access; one place to grep all handoffs; git-tracked alongside plan.json and state.json.

## Reading handoffs (planner)

For each handoff arrival:

1. **`## Status`** other than `success`: decide retry, repair, or follow-up task. (`partial` means some acceptance hit, some not; `blocked` means hard stop on something the worker couldn't resolve.)
2. **`## Branch`**: note it; record on state.json; reference it if another task needs to build on it.
3. **`## What I did`**: treat as fact; skim for claims that don't match expectations.
4. **`## Notes / concerns / deviations`**: richest section. Each bullet may become a new task. Worker feedback about scoping is especially valuable: tells you whether your prompts are pulling their weight.
5. **`## Suggested follow-ups`**: candidate tasks. Accept, reject, or consolidate.

`Status: blocked` is a single task's dead end; the planner retries, repairs, or clarifies and the tree keeps moving. Escalate to user (or "stop the line") only when continued spawning would waste effort across the tree.

## Synthetic failure handoffs

When a spawned session exits without writing its handoff (wrapper error, context-limit, network drop, tool error), the planner writes `handoffs/<task-id>-failure.md` so the trail isn't silent. Template:

```markdown
<!-- orchestrate failure handoff
task: <task-id>
branch: <actual or null>
agentId: <agent-id>
worktreePath: <path or null>
failureMode: tool-error | network-drop | context-limit | unknown | orphaned
terminatedAt: <iso-timestamp>
-->

# <task-id> — failure handoff

Status: error (worker subagent terminated without writing a handoff)
Failure mode: <mode>
Agent id: <id>
Started: <iso>
Terminated: <iso>
Duration: <ms>
Worktree: <path or null>
Branch: <branch or null>
Wrapper output summary: <truncated claude-bg-wait stderr/stdout>

## Suggested next steps
- <one bullet per option; planner picks>
```

### Classifier heuristics

Inspect the `claude-bg-wait` exit code + stderr/stdout for signals:

- `tool-error` — result contains `tool_use_failed`, `tool error`, generic SDK error stack
- `network-drop` — result contains `fetch failed`, `ETIMEDOUT`, `ECONN`, `socket`, `dns`, `disconnect`
- `context-limit` — result indicates the subagent ran out of context or hit a token cap
- `orphaned` — Phase 0 recovery couldn't find a handoff and the session was interrupted
- `unknown` — none of the above

(Cursor's `cap-hit` / `oom` cases are subsumed into `context-limit` / `tool-error` for Claude Code subagents.)

### Default retry policy

- `tool-error` → retry with different `model` (e.g. switch sonnet ↔ opus).
- `network-drop` → retry as-is (transient).
- `context-limit` → retry with narrower scope (split the task, tighten `pathsAllowed`, shorten `scopedGoal`).
- `finished-no-handoff` (see below) → read raw worker output if available; if intent is recoverable, retry with explicit "you MUST write the handoff file" reminder; else abandon.
- `unknown` → retry as-is once, then abandon.

After 2 retries on the same task, prefer abandon (drop from `plan.json`, replan around) over a 3rd attempt unless you have specific evidence the next retry will succeed.

## Finished-without-handoff sidecar

If `claude-bg-wait` exited cleanly but no handoff file exists at the expected worktree path, the worker forgot to write it. Planner writes `handoffs/<task-id>-finished-no-handoff.md`:

```markdown
<!-- orchestrate finished-no-handoff sidecar
task: <task-id>
branch: <worktreeBranch>
agentId: <agentId>
worktreePath: <worktreePath>
note: worker subagent ended without writing .orchestrate/handoffs/<task-id>.md
-->

Status: error (worker ended without structured handoff)
Branch: <branch>
Wrapper output (raw): <last ~500 chars of claude-bg-wait stdout/stderr>

## Suggested next steps
- Inspect worker's worktree at <worktreePath> for evidence of work.
- If recoverable, retry with explicit handoff-file reminder.
- If not, abandon and replan.
```

Treat the raw wrapper output as the worker's intent; decide retry or abandon. The worker's worktree still exists on disk — inspect files there for evidence.

## Upstream handoffs in downstream prompts (context relay)

Workers live on sibling worktrees and **cannot read each other's worktrees at runtime.** If a downstream task depends on an upstream task's output, the planner must relay it.

The planner pastes each upstream handoff body verbatim under `## Upstream handoffs` in the downstream spawn prompt:

```
## Upstream handoffs

### handoff: <upstream-task-id>
(verbatim body of <workspace>/handoffs/<upstream-task-id>.md)

### handoff: <another-upstream-task-id>
(verbatim body)
```

Consequences:

- `dependsOn` is semantically meaningful (context relay), not just a scheduling gate. Use it whenever a downstream task needs upstream findings, even if Git-level ordering doesn't strictly require it.
- Undeclared `dependsOn` + needed upstream context = worker guesses, drifts.
- Long fan-ins inflate prompt size. If unwieldy, publish an aggregation worker first and have it produce a summarized handoff downstream tasks consume.
- Handoffs render verbatim: sloppy `## What I did` sections pollute every downstream task. Respect the format.

## Producing your own handoff (subplanner)

A subplanner's final message is its handoff to its parent. Aggregate children upward; don't forward raw child handoffs. The parent has more global context but less local detail.

| Field | Rule |
|-------|------|
| `Status` | `success` only if every acceptance criterion is met. `partial` if any child was partial. `blocked` if any hard blocker remains. |
| `Branch` | The actual deliverable branch you are handing up. Usually the last merge-task's output within your subtree. Downstream tasks build on whatever you name here, so name the real deliverable explicitly. |
| `What my subtree did` | One bullet per meaningful slice, not per child. The parent cares about work, not your org chart. |
| `Notes / concerns` | Surface anything a sibling subtree or the root might collide with. |
| `Suggested follow-ups` | Tasks for your parent's scope, not yours. |

Subplanner template:

```markdown
## Status
success | partial | blocked

## Branch
<deliverable branch>

## What my subtree did
- <aggregated bullet>
- <aggregated bullet>

## Verification
<live-ui-verified | unit-test-verified | type-check-only | verifier-blocked | verifier-failed | not-verified>

(Aggregate the strongest claim your subtree's evidence supports. Definitions live in `prompts/verifier.md` semantics; pass `verifier-blocked` through unchanged rather than rounding up.)

## Notes, concerns, deviations, findings, thoughts, feedback
- <bubbled up from children>

## Suggested follow-ups
- <tasks for parent's scope>
```

### Subplanner respawn discipline

If a subplanner is retried after a failed attempt, **respawn from your own branch** rather than starting fresh — `git reset --hard <subplanner-branch>` so committed child state and handoffs survive the retry.

## Verifier handoffs

A verifier's final action is to write a verdict file on one target task's acceptance criteria. Format and verdict vocabulary are in the verifier spawn-prompt template (`references/spawning.md` → Verifier template). Key behavior:

- `## Verification` is parsed by the planner and persisted on the **target task's** state row (`tasks[].verification` in state.json) — not on the verifier's row. So post-run reports bucket "fixed-and-verified" properly.
- Workers and subplanners may self-report a `## Verification` line. A later verifier overrides the self-report on the same target row.
- Legacy `## Verdict pass | fail | inconclusive` migrates to: `pass` → `type-check-only`, `fail` → `verifier-failed`, `inconclusive` → `verifier-blocked`.

## Merges are tasks

Because planners don't code, merges happen via tasks. Publish a worker whose `scopedGoal` names both branches and the resolution policy:

```json
{
  "id": "merge-frontend-and-theme",
  "name": "merge-frontend-and-theme",
  "type": "worker",
  "scopedGoal": "Merge orch/dark-mode/frontend-toggle into the current branch. On conflict in packages/ui/src/settings/Settings.tsx, prefer frontend-toggle's hook wiring. After merge, verify pnpm -w typecheck passes.",
  "startingRef": "<theme-system worker's branch>",
  "dependsOn": ["frontend-toggle", "theme-system"],
  "acceptance": [
    "Merge committed with both parents in history",
    "pnpm -w typecheck passes"
  ]
}
```

Its handoff tells you whether the merge succeeded, whether conflicts were non-obvious, and whether acceptance held.

## Continuous motion

A planner isn't strictly "done" while children might still produce handoffs (most won't in a synchronous-spawn model, but late handoffs can arrive after subplanner retries or out-of-band recovery). If one arrives after you've summarized:

- Read it.
- If it changes your conclusion, say so.
- Publish follow-ups if needed.
- Produce a fresh summary.

Hard stop only when you've decided to stop publishing and every in-flight task is terminal.
