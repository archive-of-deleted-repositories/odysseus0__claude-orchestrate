# claude-orchestrate

Multi-agent orchestration for Claude Code. Decompose a large goal into a tree of parallel worker / verifier / subplanner sessions running in isolated git worktrees, collect typed handoffs, replan, repeat until terminal.

Adapted from Cursor's [`/orchestrate`](https://github.com/cursor/plugins/tree/main/orchestrate) skill — same five-primitive contract (Task / Handoff / Workspace / Roles / Loop), reimplemented for Claude Code's substrate: every spawn is a fresh top-level Claude Code session in an isolated worktree, awaited via a `claude-bg-wait` wrapper around `claude --bg --worktree`. Uniform spawn substrate; no Agent-tool subagents; daemon-supervised sessions with stable IDs queryable via `claude agents --json`.

## What it does

You type `/orchestrate:orchestrate <goal>` in Claude Code. The session becomes a **root planner**: it decomposes the goal into a `plan.json` task graph, then enters a reconcile loop. Ready tasks spawn in parallel via `Bash(claude-bg-wait <worktree-name> <prompt>)` — each call creates an isolated git worktree at `<repo>/.claude/worktrees/<name>/` and spawns a backgrounded Claude Code session in it. The session does its work, writes a handoff to `.orchestrate/handoffs/<task-id>.md` in its own worktree, exits. The wrapper detects idle-stable, returns. The planner reads the handoff file from the worktree path, copies to the central workspace, updates `state.json`, decides what to do next.

Workers, verifiers, and subplanners all use the **same** spawn substrate. The role contract for each lives in a spawn-prompt template (`skills/orchestrate/references/spawning.md`) that the planner renders per task — no separate Claude Code subagent definitions, matching Cursor's prompt-canonical design.

Sibling workers don't talk to each other. Context flows up via handoffs and down via the planner's spawn prompts (the planner pastes upstream handoff bodies verbatim into downstream prompts when `dependsOn` declares a dependency).

## Provided

- **`/orchestrate:orchestrate <goal>`** — slash command that bootstraps the planner.
- **`orchestrate:orchestrate` skill** — operating manual the planner reads (`SKILL.md` + `references/{planner,handoffs,spawning}.md` + schemas).
- **`bin/claude-bg-wait`** — PATH-accessible wrapper around `claude --bg --worktree`. Synchronous: spawns + polls + reads logs + (by default) `claude stop`s the session. Supports `--keep-session` to leave the session alive for `claude attach` debugging.
- **`bin/orchestrate-state-write`** — PATH-accessible helper for atomic JSON writes (tempfile + rename) to avoid mid-write corruption of `state.json` / `plan.json` on long runs.

No `agents/` directory. Role contracts (worker, verifier, subplanner) live in spawn-prompt templates inside `skills/orchestrate/references/spawning.md`.

## Install

### From local marketplace (development)

Clone the repo, then add to `~/.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "claude-orchestrate": {
      "source": {
        "source": "directory",
        "path": "/absolute/path/to/this/repo"
      }
    }
  },
  "enabledPlugins": {
    "orchestrate@claude-orchestrate": true
  }
}
```

Then run `claude plugin marketplace add /absolute/path/to/this/repo` and `claude plugin install orchestrate@claude-orchestrate`. Restart Claude Code. `/orchestrate:orchestrate` should appear in your slash command list, and `which claude-bg-wait` should resolve to the plugin's `bin/` (Claude Code auto-prepends every installed plugin's `bin/` to PATH).

### From GitHub (after publishing)

```json
{
  "extraKnownMarketplaces": {
    "claude-orchestrate": {
      "source": {
        "source": "github",
        "repo": "<your-username>/claude-orchestrate"
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

1. **Task** — `plan.json` entry: `scopedGoal`, `pathsAllowed`, `acceptance`, `verify` recipe, `startingRef`, `dependsOn`, `measurements`, `maxAttempts`.
2. **Handoff** — markdown file with `## Status`, `## Branch`, `## What I did`, `## Measurements`, `## Verification`, `## Notes`, `## Suggested follow-ups`. Written by the spawned session inside its own worktree at `.orchestrate/handoffs/<task-id>.md`. The planner reads from the worktree path, copies to the central `<workspace>/handoffs/`.
3. **Workspace** — `.orchestrate/<rootSlug>/` in the repo root: `plan.json`, `state.json`, `handoffs/`, `attention.log`. (Spawn prompts are inlined into the planner's Bash calls via single-quoted heredocs — no separate `spawns/` file needed; bg session logs preserve them via `claude logs <id>`.) Git-tracked (or gitignored — your call).
4. **Roles** — planner, worker, verifier, subplanner. Each is a role contract baked into a spawn-prompt template, not a separate Claude Code subagent.
5. **Loop** — in-process inside the planner's turn. Each ready task spawned via `Bash(claude-bg-wait ...)`; multiple `Bash` calls in one assistant message parallelize.

Key differences from Cursor's design:

- No dispatcher (the Claude Code session is local; no cloud handoff needed).
- No `@cursor/sdk` plumbing — every spawn is `claude --bg --worktree <name>` via the `claude-bg-wait` wrapper.
- No subprocess heartbeat (`bun cli.ts run`); the planner's own turn drives the reconcile loop.
- **Andon (cross-tree pause) is file-based**: operator touches `.orchestrate/<rootSlug>/andon` from any terminal to halt new spawns across the tree; remove the file to resume. Every planner (root + subplanner descendants) checks the root's andon file before each spawn cycle.
- **recoverRunning equivalent** exists: every spawn has a stable `sessionId` in `claude agents --json`. On parent crash, Phase 0 recovery can reattach to live in-flight sessions instead of marking them orphaned.

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

The planner auto-wires verifier `dependsOn` (the target gets added) and includes both the target's branch (`## Target`) and target's worktree path (`## Target worktree path`) in the verifier's rendered spawn prompt. Verifiers position via `cd <target-worktree-path>`; they don't reset their own branch.

## Core principles

1. **Planners own scopes and publish tasks. They do no coding.** Editing files, running `git merge`, fixing conflicts inline are worker tasks.
2. **Workers are isolated.** One task, one worktree, one branch, one handoff.
3. **`dependsOn` is semantically meaningful** — it gates spawn AND triggers context relay. The planner pastes upstream handoff bodies into downstream spawn prompts.
4. **Subplanners are recursive planners.** Same spawn substrate as workers; once spawned, the bg session reads `SKILL.md`, follows the planner contract, and can fan out its own children. No depth limit.
5. **Merges are tasks**, not planner actions.

## Pausing a long run

Operator touches the root's andon flag from any terminal:

```sh
touch .orchestrate/<rootSlug>/andon
```

Every planner in the tree checks this file before each spawn cycle. New spawns pause; in-flight work continues. To resume:

```sh
rm .orchestrate/<rootSlug>/andon
```

The planner picks up the loop on its next sweep tick.

## Roadmap

- **v0.1 (this release):** the five primitives, uniform `claude --bg` spawn substrate, recursive subplanners, file-canonical handoffs, file-based Andon, atomic state writes, verifier `cd`-into-target auto-wiring, `bin/`-on-PATH plugin packaging.
- **v0.1.1+:** harden the `recoverRunning` path on real parent-crash; stress-test `maxAttempts` retry on a real synthetic failure; exercise `measurements[]` rerun.
- **v0.2:** dedicated TUI dashboard for tree visualization (the orchestrate tree is naturally hierarchical; `claude agents --json` shows the flat list, but a tree view would help operators).
- **Future:** multi-machine workspace (workers on remote infrastructure via separate Claude Code sessions sharing a workspace via git push/pull).

## Credits

Design and primitive contracts adapted from [Cursor's `/orchestrate` plugin](https://github.com/cursor/plugins/tree/main/orchestrate). All credit to that team for the substrate-agnostic five-primitive design; this port reimplements it on Claude Code's substrate without altering the conceptual surface.

## License

MIT — see [LICENSE](./LICENSE).
