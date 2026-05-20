---
description: "Decompose a large goal into a tree of parallel worker/verifier/subplanner tasks. Spawns workers in isolated worktrees, dispatches recursive subplanners via daemon-supervised bg sessions, collects structured handoffs, drives the reconcile loop until terminal."
argument-hint: "<goal>"
---

You are the root planner for: $ARGUMENTS

Read `.claude/skills/orchestrate/SKILL.md` and follow it.

Discover here before you publish tasks (read existing repo state, scan for relevant code). Workers and subplanners spawn with no shared memory of what you saw — bake the essentials into their spawn prompts.

Loop hygiene:
- Run the reconcile loop in-process inside this turn. Do not detach. The loop IS your turn.
- After publishing tasks and surfacing the plan for user approval, enter the reconcile loop: spawn ready tasks (workers/verifiers via Agent tool; subplanners via `Bash(claude-bg-wait …)`), wait for handoffs, persist state, repeat until no pending task is reachable.
- On any task transitioning to `error`: read the synthetic failure handoff at `<workspace>/handoffs/<task>-failure.md`, decide retry / split / escalate / drop, then continue. Each retry costs another spawn — after 2 attempts on the same task, prefer abandon (drop from plan.json, replan around it) over a 3rd attempt unless you have specific evidence the next retry will succeed.
- Don't end your turn while this workspace has non-terminal tasks.
