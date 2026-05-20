---
name: orchestrate
description: "Use only when the user explicitly types `/orchestrate:orchestrate <goal>` to decompose a large task, spawn a tree of parallel worker/subplanner/verifier subagents, and collect structured handoffs. Do not invoke autonomously."
---

# Orchestrate

An explicit `/orchestrate:orchestrate <goal>` fans out a large task across parallel Claude Code subagents running in isolated git worktrees. Workers don't talk to each other; they talk up through structured handoffs. The spawn, wait, and handoff loop lives in the planner's own turn. The planner writes `plan.json`, drives the loop, reads handoffs to decide what comes next. Long-running agent loops drift; a script with a JSON state file keeps its footing — in Claude Code the "script" is the planner's structured procedure in `references/planner.md`.

Adapted from Cursor's `/orchestrate` skill ([cursor/plugins/orchestrate](https://github.com/cursor/plugins/tree/main/orchestrate)). Substrate:

- No dispatcher. The Claude Code session is local; it IS the root planner.
- **Uniform spawn substrate.** Every spawn (worker, verifier, subplanner) is a fresh top-level Claude Code session in its own isolated git worktree, spawned via `claude --bg --worktree <name>` and awaited via the bundled `claude-bg-wait` wrapper (on PATH via `<plugin>/bin/`). Sessions are daemon-supervised, have stable ids visible in `claude agents --json`, and can be re-bound to after parent crashes (recoverRunning equivalent).
- No Slack. **Andon (cross-tree pause) is file-based:** the operator touches `.orchestrate/<rootSlug>/andon` from any terminal to halt new spawns across the tree; removing the file resumes. Every planner (root + all subplanner descendants) checks the root's andon file before each spawn cycle. See `references/planner.md` Phase 2 sweep step 0.
- No subprocess heartbeat; the planner's own turn drives the loop.

## Core principles

These rules make the tree self-converging without global coordination.

1. **Planners own scopes and publish tasks. They do no coding.** Writing `plan.json`, reading handoffs, and deciding what's next are planner work. Editing files, running `git merge`, and fixing conflicts inline are not. If a planner feels the urge to code, it publishes a task for a worker instead.
2. **Planners don't know who picks up their tasks.** Routing each task to a fresh top-level Claude Code session (via `claude-bg-wait <worktree-name> <prompt>`) is mechanical. The planner's mental model stays at the task level.
3. **Workers are isolated.** One task, one worktree, no channel to any other agent. One handoff when done.
4. **Subplanners are recursive planners.** A planner publishes a "subplan this slice" task; the subplanner fully owns that slice and hands back an aggregated handoff. Subplanners can spawn their own subplanners — no depth limit.
5. **Continuous motion via handoffs.** A planner that thought it was done can receive a late handoff and replan. No "finished" state until the planner decides to stop publishing.
6. **Propagation, not synchronization.** No cross-talk between siblings. No shared state between levels. Each level sees only its children's handoffs.

## Node types

| Node           | Spawn substrate | Scope                            | Output                                  |
| -------------- | --------------- | -------------------------------- | --------------------------------------- |
| Root planner   | `/orchestrate:orchestrate <goal>` slash command (user-initiated; root runs in the user's session) | Entire user goal | User-facing summary |
| Subplanner (↻) | `Bash(claude-bg-wait <worktree-name> <prompt>)` | One slice of parent's scope | Handoff file in its worktree |
| Worker         | `Bash(claude-bg-wait <worktree-name> <prompt>)` | One concrete task | Handoff file in its worktree |
| Verifier       | `Bash(claude-bg-wait <worktree-name> <prompt>)` | One target's acceptance criteria | Verdict handoff file in its worktree |
| Git            | n/a             | Shared medium                    | Branches (code) + handoffs/ (meaning)   |

## What to read next

Read `references/planner.md` — the full operating manual for root and subplanners. It is the load-bearing read.

Other references:
- `references/handoffs.md` — handoff schemas, synthetic failures, merges-as-tasks, upstream context relay.
- `references/spawning.md` — spawn-prompt contracts for workers/verifiers/subplanners, including the subplanner template you render and pass to `claude-bg-wait`.
- `schemas/plan.schema.json` + `schemas/state.schema.json` — task graph and state row schemas.
