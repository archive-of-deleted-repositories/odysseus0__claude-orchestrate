# Spawning

The contracts between planner and the three spawn substrates (Agent tool for workers/verifiers; `claude-bg-wait` for subplanners).

## Workers and verifiers: Agent tool

```
Agent(
  subagent_type = "orchestrate:orchestrate-worker" | "orchestrate:orchestrate-verifier",
  isolation     = "worktree",
  description   = "<rootSlug>/<task-id>",      # appears in spinner / logs
  prompt        = <rendered spawn prompt>,
)
```

Multiple ready tasks: emit multiple `Agent` tool calls in a single assistant message. They run in parallel.

### Agent-tool worktree behavior

`isolation: "worktree"` causes Claude Code to:
- Create a worktree at `~/.claude/worktrees/agent-<id>/`.
- Set the subagent's `pwd` to that worktree.
- Auto-name the branch `worktree-agent-<id>`.
- On agent exit: persist worktree + branch if changes were made; auto-clean if not.

Result format (after the prose):

```
agentId: <id> (use SendMessage with to: '<id>' to continue this agent)
worktreePath: /Users/.../agent-<id>
worktreeBranch: worktree-agent-<id>
```

Parse these lines. Persist `agentId`, `worktreePath`, `worktreeBranch` to state.json for the task.

### Worker/verifier spawn prompt template

```
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

## Verify recipe
<task.verify markdown, if present>

## Starting ref
<task.startingRef, only if present — worker `git reset --hard`s to this as first action>

## Measurements
- <m.name>: `<m.command>`
(only if task.measurements is non-empty; otherwise omit the section)

## Upstream handoffs
### handoff: <dep-id>
<verbatim handoff body>

### handoff: <other-dep-id>
<verbatim handoff body>
(only if task.dependsOn has entries with status: handed-off; otherwise omit)

## Task id
<task.id>
```

The worker / verifier system prompt (loaded automatically when the subagent type is invoked) tells the subagent what to do with this content — including writing the handoff to `.orchestrate/handoffs/<task-id>.md` in its own worktree.

## Subplanners: `claude-bg-wait`

```
Bash(
  command = "claude-bg-wait \"$(cat <workspace>/spawns/<task-id>.md)\""
)
```

`claude-bg-wait` is bundled in this plugin's `scripts/` directory. It spawns `claude --bg <prompt>`, polls until the bg session is idle-stable, prints session logs to stdout, then `claude stop`s the session. The bg session is a fresh top-level Claude Code session — full Agent tool access (so it can recursively spawn its own workers + further subplanners).

**The bg session's stdout from `claude logs` is noisy** (raw TUI output with ANSI escapes). Don't try to parse it for the handoff. Instead, the subplanner spawn prompt tells the subplanner to write its handoff to a file path you specify, and you `Read` that file after `claude-bg-wait` returns.

### Subplanner spawn prompt template

Write to `<workspace>/spawns/<task-id>.md` before invoking `claude-bg-wait`:

```
You are a subplanner for: <task.scopedGoal>

Read `.claude/skills/orchestrate/SKILL.md` and follow it.

**You fully own this slice.** Your parent gave you a goal, path boundaries, and acceptance, not a sub-plan. Decide your own decomposition. If `scopedGoal` below includes hints about how to split the work, treat them as weak hints at most; you are authoritative on your subtree's structure.

**Recursion.** You are a planner: use workers for leaf-sized slices; add `subplanner` tasks when the slice still needs internal structure or merge/verify passes. No depth limit. Use judgment on count: one worker can carry a multi-file, multi-step slice. Default to fewer, broader workers; see `references/planner.md` "Planning rules" for the spawn-scope tradeoff.

**Child tasks.** Write child tasks as `plan.tasks[]` entries in your own plan.json. For code-editing children, propagate per the code discipline: no narrative comments. Comment only non-obvious *why*.

**Workspace convention.** Put your orchestrate workspace at `.orchestrate/<task.name>/` and set `plan.rootSlug = "<task.name>"`. Your cwd is the parent's cwd — you can see the parent's `.orchestrate/<parent-rootSlug>/` but should not modify it; carve out your own.

Overall goal (parent's framing, context only):

<plan.goal>

Your scoped sub-goal:

<task.scopedGoal>

Paths you may MODIFY (read any file in the repo):
<task.pathsAllowed>

Paths you must NOT modify (owned by siblings):
<task.pathsForbidden>

Acceptance criteria for your subtree:
<task.acceptance>

## Upstream handoffs
### handoff: <dep-id>
<verbatim handoff body>
(only if task.dependsOn has entries with status: handed-off; otherwise omit)

## Handoff file path
When you finish, write your aggregated handoff to: `<absolute-path-to-parent-workspace>/handoffs/<task.id>.md`

This file path is in the parent planner's workspace, which is in your cwd. The parent reads from this exact path after you exit. Do not write to anywhere else for the handoff.

## Task id
<task.id>

Your **final action** is to write your handoff to the path above. Use the subplanner handoff structure documented in `references/planner.md` (Phase 3 — Subplanner). Don't end your turn until that file is on disk.
```

The bg session reads this entire prompt, treats it as its initial user message, loads the orchestrate skill via the "Read .claude/skills/orchestrate/SKILL.md and follow it" instruction, and proceeds.

## `startingRef` (worker-side switch)

The Agent tool creates the worktree from current HEAD (the planner's branch). If a worker/verifier task declares `startingRef: <ref>`, the spawn prompt includes:

```
## Starting ref
<ref>
```

The worker's subagent system prompt instructs it to run `git reset --hard <ref>` as its first action — repointing its branch's HEAD from the planner's HEAD to the desired ref. The branch name doesn't change; the content does.

Subplanners don't go through this mechanism — they share the planner's cwd and HEAD. If a subplanner needs to start from a specific ref, its scopedGoal should say so and the subplanner can `git reset --hard` in its own worktree(s) when it spawns its workers.

## `dependsOn` (gating + context relay)

`dependsOn: [<task-id>, ...]` does two things:
1. **Gates spawn**: a task with `dependsOn` cannot become ready until every listed task has `status: handed-off`.
2. **Triggers context relay**: at spawn time, the planner pastes each listed task's handoff body verbatim into the downstream spawn prompt under `## Upstream handoffs`.

Use `dependsOn` whenever a downstream task needs upstream context, even if Git-level ordering doesn't strictly require it. Workers can't read sibling worktrees at runtime.

## Verifier auto-wiring

When the planner encounters a task with `type: verifier` and `verifies: <target-id>`:
- **Implicit `dependsOn`**: target id is added to verifier's `dependsOn` at plan-load time.
- **Implicit `startingRef`**: once target hands off, planner sets verifier's `startingRef` to `<target>.branch` from state.json before spawning.

The verifier then `git reset --hard <target-branch>` as its first action and verifies on the actual delivered code.

## State persistence per task

After each spawn returns, the planner updates `state.json` for that task with:

```json
{
  "id": "<task-id>",
  "status": "handed-off" | "error" | "partial" | "blocked" | "cancelled",
  "agentId": "<id from result, for workers/verifiers; for subplanners use bg session id>",
  "worktreePath": "<path from result, for workers/verifiers; null for subplanners>",
  "branch": "<worktreeBranch from result, for workers/verifiers; from handoff '## Branch' line for subplanners>",
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
