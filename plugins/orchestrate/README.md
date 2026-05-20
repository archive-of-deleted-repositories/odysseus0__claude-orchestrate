# orchestrate

Multi-agent orchestration plugin for Claude Code. See the [top-level README](../../README.md) for the full design rationale, architecture, and install instructions.

## Files in this plugin

```
plugins/orchestrate/
├── .claude-plugin/
│   └── plugin.json           # plugin manifest
├── bin/
│   ├── claude-bg-wait        # synchronous wrapper around `claude --bg --worktree`
│   │                         # accepts [--keep-session] to skip the final `claude stop`
│   └── orchestrate-state-write  # atomic JSON write helper (tempfile + rename)
├── commands/
│   └── orchestrate.md        # /orchestrate:orchestrate slash command
└── skills/
    └── orchestrate/
        ├── SKILL.md          # top-level skill entry
        ├── references/
        │   ├── planner.md    # operating manual for root and subplanners
        │   ├── handoffs.md   # handoff schemas + synthetic failures + context relay
        │   └── spawning.md   # spawn-prompt templates (worker / verifier / subplanner)
        └── schemas/
            ├── plan.schema.json
            └── state.schema.json
```

**No `agents/` directory.** Every spawn (worker, verifier, subplanner) is a fresh top-level Claude Code session in its own isolated worktree, awaited via the `claude-bg-wait` wrapper. Role contracts live entirely in spawn-prompt templates in `skills/orchestrate/references/spawning.md`, matching Cursor's prompt-canonical design.

## Quick usage

After install, type `/orchestrate:orchestrate <goal>` in any Claude Code session. The session becomes the root planner. It will publish a task plan, pause for your approval, then drive the reconcile loop until terminal.

The planner does no coding — every file modification happens inside a worker task. Workers run in isolated git worktrees (`<repo>/.claude/worktrees/<worktree-name>/`) and communicate up via structured handoff files. See `skills/orchestrate/references/planner.md` for the load-bearing operating manual.

To pause a long-running orchestrate tree without killing the planner session, `touch .orchestrate/<rootSlug>/andon` from any terminal. The planner checks this file before each spawn cycle; remove it to resume. See `skills/orchestrate/references/planner.md` Phase 2 sweep step 0.

To debug a spawned session, pass `--keep-session` to the wrapper invocation:

```sh
claude-bg-wait --keep-session <worktree-name> <prompt>
```

The session will stay listed in `claude agents` after the wrapper returns; attach with `claude attach <agent_id>`.
