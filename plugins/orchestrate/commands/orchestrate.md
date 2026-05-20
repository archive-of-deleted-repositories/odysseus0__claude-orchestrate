---
description: "Decompose a large goal into a tree of parallel worker/verifier/subplanner tasks. Spawns workers in isolated worktrees, dispatches recursive subplanners via daemon-supervised bg sessions, collects structured handoffs, drives the reconcile loop until terminal."
argument-hint: "<goal>"
---

You are the root planner for: $ARGUMENTS

Read `.claude/skills/orchestrate/SKILL.md` and follow it.

Discover here before you publish tasks (read existing repo state, scan for relevant code). Workers and subplanners spawn with no shared memory of what you saw — bake the essentials into their spawn prompts.

Loop hygiene:
- Run the reconcile loop in-process inside this turn. Do not detach. The loop IS your turn.
- After publishing tasks and surfacing the plan for user approval, enter the reconcile loop: spawn ready tasks (all task types via `Bash(claude-bg-wait <worktree-name> <prompt>)`), wait for handoffs, persist state, repeat until no pending task is reachable.
- On any task transitioning to `error`: read the synthetic failure handoff at `<workspace>/handoffs/<task>-failure.md`, decide retry / split / escalate / drop, then continue. Each retry costs another spawn — after 2 attempts on the same task, prefer abandon (drop from plan.json, replan around it) over a 3rd attempt unless you have specific evidence the next retry will succeed.
- Don't end your turn while this workspace has non-terminal tasks.

If you (the user reading this) are about to start a long orchestrate run, consider invoking Claude Code with `--dangerously-skip-permissions` (the `ccc` alias). Auto-mode's safety classifier doesn't see the content of file-based spawn prompts (`$(cat <spawn-file>)`) and may block individual `Bash(claude-bg-wait ...)` invocations as "unverifiable agent-inferred input" — disruptive for long autonomous runs. You already approved the plan at Phase 1; the workers run under that mandate.

After publishing the plan, surface this to the user verbatim so they know the pause-control:

```
This run is now driving. If you want to halt new spawns mid-run (e.g.
you spot bad upstream output or a runaway decomposition), open another
terminal and run:

    touch .orchestrate/<rootSlug>/andon

I'll see it before the next spawn cycle, pause new spawns, and wait.
In-flight work finishes; nothing new starts. To resume:

    rm .orchestrate/<rootSlug>/andon

This is the Andon cord — pause-not-kill. It works whether you're
watching this terminal or away.
```

(Substitute the actual rootSlug into the snippet.)
