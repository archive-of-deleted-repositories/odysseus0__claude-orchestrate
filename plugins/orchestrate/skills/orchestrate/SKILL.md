---
name: orchestrate
description: "Use only when the user explicitly types `/orchestrate <goal>` to decompose a large task, spawn a tree of parallel worker/subplanner/verifier subagents via Claude Code's Agent tool, and collect structured handoffs. Do not invoke autonomously."
---

# Orchestrate

An explicit `/orchestrate <goal>` fans out a large task across parallel Claude Code subagents running in isolated git worktrees. Workers don't talk to each other; they talk up through structured handoff files. The spawn/wait/handoff loop runs inside the planner's own session — no subprocess.

Adapted from Cursor's `/orchestrate` skill ([cursor/plugins/orchestrate](https://github.com/cursor/plugins/tree/main/orchestrate)). Key differences from Cursor's design:

- No dispatcher; the Claude Code session is local, so it directly becomes the planner.
- No `@cursor/sdk` and no cloud VMs; workers spawn via the `Agent` tool with `isolation: "worktree"`, which manages worktrees at `~/.claude/worktrees/agent-<id>/`.
- No subprocess heartbeat (`bun cli.ts run`); the planner's own turn drives the reconcile loop.
- No Slack mirroring or Andon polling; the user is in-session and Ctrl-C is the emergency stop.

Workspace layout (after kickoff):

```
.orchestrate/<rootSlug>/
├── plan.json          (planner-authored task graph)
├── state.json         (planner-authored per-task state)
├── handoffs/          (copied from worker worktrees; canonical history)
└── attention.log      (drift signals, surfaced to user live)
```

Worker worktrees live at `~/.claude/worktrees/agent-<id>/` (Claude Code-managed).

## Core principles

These rules make the tree self-converging without global coordination.

1. **Planners own scopes and publish tasks. They do no coding.** Writing `plan.json`, reading handoffs, deciding what's next are planner work. Editing files, running `git merge`, fixing conflicts inline are not. If a planner feels the urge to code, it publishes a task for a worker instead.
2. **Planners don't know who picks up their tasks.** The `Agent` tool routes each task to a subagent (a worker, verifier, or subplanner).
3. **Workers are isolated.** One task, one worktree, no channel to any other agent. One handoff when done.
4. **Subplanners are recursive planners.** A planner publishes a "subplan this slice" task; the subplanner fully owns that slice and hands back an aggregated handoff.
5. **Continuous motion via handoffs.** A planner that thought it was done can still receive a late handoff and replan.
6. **Propagation, not synchronization.** No cross-talk between siblings. No shared state between levels. Each level sees only its children's handoffs.

## Node types

| Node           | Runs the loop? | Scope                            | Output                                 |
| -------------- | -------------- | -------------------------------- | -------------------------------------- |
| Root planner   | yes            | Entire user goal                 | User-facing summary                    |
| Subplanner (↻) | yes (recursive)| One slice of parent's scope      | Handoff to parent                      |
| Worker         | no             | One concrete task                | Handoff to spawning planner            |
| Verifier       | no             | One target's acceptance criteria | Verdict handoff to spawning planner    |
| Git            | n/a            | Shared medium                    | Branches (code) + handoffs/ (meaning)  |

## What to read next

Two roles. Read your role's reference and skip the other.

- **Planner (root or sub).** You were spawned via the `/orchestrate:orchestrate` slash command (root) or via an `Agent` tool call with `subagent_type: "orchestrate:orchestrate-planner"` and a scoped goal (sub). Read `references/planner.md`. It is the load-bearing read.
- **Worker / Verifier.** You were spawned via Agent tool with `subagent_type: "orchestrate:orchestrate-worker"` or `orchestrate:orchestrate-verifier`. Your spawn prompt is self-contained; you don't need this skill. (Your subagent system prompt — this plugin's `agents/orchestrate-worker.md` or `agents/orchestrate-verifier.md` — already contains the operational guidance.)

References:

- `references/planner.md` — operating manual for planners (root and sub).
- `references/handoffs.md` — handoff schemas, synthetic failures, merges-as-tasks, upstream context relay.
- `references/spawning.md` — spawn prompt contract, branch naming, startingRef + dependsOn semantics, verifier auto-wiring.
- `schemas/plan.schema.json` + `schemas/state.schema.json` — task graph and state row schemas.
