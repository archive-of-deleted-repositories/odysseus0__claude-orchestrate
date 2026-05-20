# claude-orchestrate

Multi-agent orchestration for Claude Code. Decompose a large goal into a tree of parallel worker / verifier / subplanner subagents running in isolated git worktrees, collect typed handoffs, replan, repeat until terminal.

Adapted from Cursor's [`/orchestrate`](https://github.com/cursor/plugins/tree/main/orchestrate) skill — same five-primitive contract (Task / Handoff / Workspace / Roles / Loop), reimplemented idiomatically with Claude Code's `Agent` tool, plugin subagents, and built-in worktree isolation. No cloud VMs, no SDK plumbing, no Slack mirroring; the planner's own session is the heartbeat.

## What it does

You type `/orchestrate:orchestrate <goal>` in Claude Code. The session becomes a **root planner**: it decomposes the goal into a `plan.json` task graph, then enters a reconcile loop. Ready tasks spawn in parallel via the `Agent` tool with `isolation: "worktree"` — each gets its own git worktree at `~/.claude/worktrees/agent-<id>/` and its own branch. Workers write structured handoff files; the planner reads them from worktree paths, copies them to a central `<workspace>/handoffs/` directory, updates `state.json`, and decides what to do next.

Workers don't talk to each other. Context flows up via handoffs and down via the planner's spawn prompts (the planner pastes upstream handoff bodies verbatim into downstream prompts when `dependsOn` declares a dependency).

## Provided

- `/orchestrate:orchestrate <goal>` — slash command that bootstraps the planner.
- `orchestrate:orchestrate` skill — operating manual the planner reads.
- `orchestrate:orchestrate-planner` subagent — root planner / subplanner contract.
- `orchestrate:orchestrate-worker` subagent — one-shot worker contract.
- `orchestrate:orchestrate-verifier` subagent — read-only verifier contract.

## Install

### From local marketplace (development)

Add to `~/.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "claude-orchestrate": {
      "source": {
        "source": "local",
        "path": "/absolute/path/to/this/repo"
      }
    }
  },
  "enabledPlugins": {
    "orchestrate@claude-orchestrate": true
  }
}
```

Restart Claude Code. `/orchestrate:orchestrate` should appear in your slash command list.

### From GitHub (after release)

```json
{
  "extraKnownMarketplaces": {
    "claude-orchestrate": {
      "source": {
        "source": "github",
        "repo": "georgezhangtj/claude-orchestrate"
      }
    }
  },
  "enabledPlugins": {
    "orchestrate@claude-orchestrate": true
  }
}
```

## Architecture (TL;DR)

The five primitives the port realizes (substrate-agnostic, lifted from Cursor's design):

1. **Task** — `plan.json` entry: scopedGoal, pathsAllowed, acceptance, verify recipe, startingRef, dependsOn, measurements, maxAttempts.
2. **Handoff** — markdown file with `## Status`, `## Branch`, `## What I did`, `## Measurements`, `## Verification`, `## Notes`, `## Suggested follow-ups`. Written by worker in its worktree; read by planner via the Agent tool's returned `worktreePath`; copied to central `<workspace>/handoffs/`.
3. **Workspace** — `.orchestrate/<rootSlug>/` in the repo root: `plan.json`, `state.json`, `handoffs/`, `attention.log`. Git-tracked.
4. **Roles** — planner, worker, verifier, subplanner. Each is a separate Claude Code subagent type with its own system prompt + tool whitelist.
5. **Loop** — in-process inside the planner's turn. No subprocess. `Agent` tool calls block until each subagent exits; multiple calls in one assistant message run in parallel.

Key differences from Cursor's design:

- No dispatcher (the Claude Code session is local; no cloud handoff needed).
- No `@cursor/sdk` plumbing — workers spawn via Claude Code's `Agent` tool with `isolation: "worktree"`.
- No subprocess heartbeat (`bun cli.ts run`); the planner's own turn drives the reconcile loop.
- No Slack mirroring or Andon polling; the user is in-session and Ctrl-C is the emergency stop.
- No `recoverRunning` via SDK reattach; if a session dies mid-run, resume reads disk state and inspects worktrees for committed handoffs.

## Authoring tasks

Inside the planner's `plan.json`:

```json
{
  "goal": "Verbatim user goal.",
  "rootSlug": "dark-mode",
  "baseBranch": "main",
  "tasks": [
    {
      "id": "frontend-toggle",
      "type": "worker",
      "scopedGoal": "Add a Settings UI toggle for dark mode...",
      "pathsAllowed": ["packages/ui/src/settings/**"],
      "acceptance": ["Toggle renders in Settings", "..."],
      "maxAttempts": 2
    },
    {
      "id": "verify-toggle",
      "type": "verifier",
      "verifies": "frontend-toggle",
      "scopedGoal": "Verify the toggle works...",
      "acceptance": ["Verification section includes execution evidence"]
    }
  ]
}
```

The planner auto-wires verifier `startingRef` to the target's branch and adds the target to its `dependsOn` at spawn time.

## Core principles

1. **Planners own scopes and publish tasks. They do no coding.** Editing files, running `git merge`, fixing conflicts inline are worker tasks.
2. **Workers are isolated.** One task, one worktree, no channel between siblings. One handoff when done.
3. **`dependsOn` is semantically meaningful** — it gates spawn AND triggers context relay. The planner pastes upstream handoff bodies into downstream spawn prompts.
4. **Subplanners are recursive planners.** Same contract; aggregate child handoffs upward.
5. **Merges are tasks**, not planner actions.

## Roadmap

- v0.1 (this release): the five primitives, single-shot reconcile loop, file-canonical handoffs.
- v0.2: `.orchestrate/<rootSlug>/andon` flag file for cross-tree pause; v0.1 relies on Ctrl-C.
- v0.2: Dedicated TUI dashboard for tree visualization.
- Future: multi-machine workspace (workers run on remote infrastructure via separate Claude Code sessions).

## Credits

Design and primitive contracts adapted from [Cursor's `/orchestrate` plugin](https://github.com/cursor/plugins/tree/main/orchestrate). All credit to that team for the substrate-agnostic five-primitive design.

## License

MIT — see [LICENSE](./LICENSE).
