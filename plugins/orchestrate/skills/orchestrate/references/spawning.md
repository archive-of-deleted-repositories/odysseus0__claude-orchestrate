# Spawning

The contract between planner and Agent tool invocation. Used when the planner spawns workers, verifiers, or subplanners.

## Agent tool call shape

```
Agent(
  subagent_type = "orchestrate:orchestrate-worker" | "orchestrate:orchestrate-verifier" | "orchestrate:orchestrate-planner",
  isolation     = "worktree",
  description   = "<rootSlug>/<task-id>",     # appears in spinner / logs
  prompt        = <rendered spawn prompt>,
)
```

Multiple ready tasks: emit multiple `Agent` tool calls in a single assistant message. They run in parallel — verified empirically (two ~10s concurrent spawns in spike tests).

## Worktree behavior (Claude Code-managed)

`isolation: "worktree"` causes Claude Code to:

- Create a worktree at `~/.claude/worktrees/agent-<id>/`.
- Set the subagent's `pwd` to that worktree.
- Auto-name the branch `worktree-agent-<id>`.
- On agent exit:
  - If the agent made zero changes, auto-clean the worktree.
  - Otherwise, persist it and return metadata.

Result format (after the prose):

```
agentId: <id> (use SendMessage with to: '<id>' to continue this agent)
worktreePath: /Users/.../agent-<id>
worktreeBranch: worktree-agent-<id>
```

Parse these lines. Persist `agentId`, `worktreePath`, `worktreeBranch` to state.json for the task.

The auto-generated branch name is what it is; **don't ask workers to rename branches**. Cursor uses `orch/<rootSlug>/<task>` because the SDK leaves the branch unset; Claude Code returns the actual branch unambiguously, so naming convention is unnecessary.

## `startingRef` (worker-side switch)

The Agent tool creates the worktree from current HEAD (the planner's branch). If a task declares `startingRef: <ref>`, the spawn prompt includes:

```
## Starting ref
<ref>
```

The worker's subagent system prompt instructs it to run `git reset --hard <ref>` as its first action — repointing its branch's HEAD from the planner's HEAD to the desired ref. The branch name doesn't change; the content does.

This preserves real parallelism for dependency chains:

- B and C both depend on A. Once A hands off, planner can spawn B and C in parallel — each with `startingRef = A.branch`. Both worktrees `git reset --hard A.branch`, both work in parallel.
- No planner-side `git merge` required before spawn.

## `dependsOn` (gating + context relay)

`dependsOn: [<task-id>, ...]` does two things:

1. **Gates spawn**: a task with `dependsOn` cannot become ready until every listed task has `status: handed-off`.
2. **Triggers context relay**: at spawn time, the planner pastes each listed task's handoff body verbatim into the downstream spawn prompt under `## Upstream handoffs`. See `handoffs.md` → "Upstream handoffs in downstream prompts".

Use `dependsOn` whenever a downstream task needs upstream context, even if Git-level ordering doesn't strictly require it. Workers can't read sibling worktrees at runtime.

## Verifier auto-wiring

When the planner encounters a task with `type: verifier` and `verifies: <target-id>`:

- **Implicit `dependsOn`**: target id is added to verifier's `dependsOn` at plan-load time (or at first spawn-readiness check). Planner doesn't require the user to wire this manually.
- **Implicit `startingRef`**: once target hands off, planner sets verifier's `startingRef` to `<target>.branch` from state.json before spawning. (If the target's branch is unknown at plan time, this happens at sweep time.)

The verifier then `git reset --hard <target-branch>` as its first action and verifies on the actual delivered code.

## Spawn prompt template

The canonical shape the planner renders for each task:

```
<role-specific header>

## Overall goal
<plan.goal verbatim — for context, not ownership>

## Your scoped task
<task.scopedGoal verbatim>

## Paths you may modify
<task.pathsAllowed, one per line, or "(no restriction)">

## Paths you must NOT modify
<task.pathsForbidden, one per line, or "(none)">

## Acceptance criteria
- <criterion 1>
- <criterion 2>
...

## Verify recipe
<task.verify markdown, if present>

## Starting ref
<task.startingRef, only if present>

## Measurements
- <m.name>: `<m.command>`
(only if task.measurements is non-empty; otherwise omit)

## Upstream handoffs
### handoff: <dep-id>
<verbatim handoff body>

### handoff: <other-dep-id>
<verbatim handoff body>
(only if task.dependsOn has entries with status: handed-off; otherwise omit)

## Task id
<task.id>
```

### Role-specific headers

- **Worker**: "You are a worker in an orchestrated task. You do not communicate with any other agents. You produce one handoff when done."
- **Verifier**: "You are a verifier for target task `<target-id>` on branch `<target-branch>`. You do not communicate with any other agents. You produce one verdict handoff when done."
- **Subplanner**: "You are a subplanner for: `<scopedGoal>`. You fully own this slice. Load the orchestrate skill (`orchestrate:orchestrate`) and read `references/planner.md`. Set `plan.rootSlug = <suggested-sub-slug>`. Aggregate child handoffs upward."

Each role's subagent system prompt — bundled with the plugin as `agents/orchestrate-<role>.md` and addressable as `orchestrate:orchestrate-<role>` — contains the operational guidance for what to do with this prompt: handoff structure, exit discipline, etc.

## Handoff file location

The worker writes its handoff to `.orchestrate/handoffs/<task-id>.md` **inside its own worktree**. The central workspace path `.orchestrate/<rootSlug>/handoffs/` does NOT exist in the worker's worktree (the worker's worktree starts from HEAD with no `.orchestrate/<rootSlug>/` directory — or if there is one, it's the planner's `.orchestrate/`, which the worker should not touch).

Convention: the worker writes to `.orchestrate/handoffs/<task-id>.md` (a top-level handoffs dir, no rootSlug nesting) inside its worktree. The planner reads from that path and copies to the central `<workspace>/handoffs/<task-id>.md`.

## State persistence per task

After each spawn returns, the planner updates `state.json` for that task with:

```json
{
  "id": "<task-id>",
  "status": "handed-off" | "error" | "partial" | "blocked" | "cancelled",
  "agentId": "<id from result>",
  "worktreePath": "<path from result>",
  "branch": "<worktreeBranch from result>",
  "attempts": <incremented>,
  "startedAt": "<iso>",
  "finishedAt": "<iso>",
  "verification": "<from handoff Verification field, if present>",
  "handoffPath": "<workspace>/handoffs/<task-id>.md",
  "failureMode": "<set on error>",
  "note": "<short operator-readable, if any>"
}
```

For verifiers, also update the **target task's** state row with the verifier's `verification` value (overrides any worker self-report).

## Quality floor

The worker's subagent system prompt enforces:
- No placeholder TODOs or `not implemented` throws.
- No narrative comments (comment only non-obvious *why*).
- For UI / interactive bugs: capture artifacts and reference paths in the handoff.

Planner doesn't need to repeat these in every spawn prompt — the worker's role agent already says them.
