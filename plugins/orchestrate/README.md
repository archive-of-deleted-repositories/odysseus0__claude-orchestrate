# orchestrate

Multi-agent orchestration plugin for Claude Code. See the [top-level README](../../README.md) for the full design rationale and architecture.

## Files in this plugin

```
plugins/orchestrate/
├── .claude-plugin/
│   └── plugin.json           # plugin manifest
├── bin/
│   ├── claude-bg-wait        # synchronous wrapper around `claude --bg --worktree`
│   └── orchestrate-state-write  # atomic JSON write helper (tempfile + rename)
├── commands/
│   └── orchestrate.md        # /orchestrate:orchestrate slash command
└── skills/
    └── orchestrate/
        ├── SKILL.md          # top-level skill entry
        ├── references/
        │   ├── planner.md    # operating manual for root and subplanners
        │   ├── handoffs.md   # handoff schemas + synthetic failures + relay
        │   └── spawning.md   # spawn-prompt templates (worker / verifier / subplanner)
        └── schemas/
            ├── plan.schema.json
            └── state.schema.json
```

No `agents/` directory — every spawn is a fresh top-level Claude Code session in its own isolated worktree, spawned via `claude --bg --worktree <name>` and awaited via `claude-bg-wait`. Role contracts live entirely in the spawn-prompt templates in `skills/orchestrate/references/spawning.md`, matching Cursor's prompt-canonical design.

## Quick usage

After install, type `/orchestrate:orchestrate <goal>` in any Claude Code session. The session becomes the root planner. It will publish a task plan, pause for your approval, then drive the reconcile loop until terminal.

The planner does no coding — every file modification happens inside a worker task. Workers run in isolated git worktrees and communicate up via structured handoff files. See `skills/orchestrate/references/planner.md` for the load-bearing operating manual.

To pause a long-running orchestrate tree without killing the planner session, `touch .orchestrate/<rootSlug>/andon` from any terminal. The planner checks this file before each spawn cycle; remove it to resume. See `skills/orchestrate/references/planner.md` Phase 2 sweep step 0.
