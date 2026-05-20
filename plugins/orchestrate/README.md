# orchestrate

Multi-agent orchestration plugin for Claude Code. See the [top-level README](../../README.md) for the full design rationale and architecture.

## Files in this plugin

```
plugins/orchestrate/
├── .claude-plugin/
│   └── plugin.json           # plugin manifest
├── agents/
│   ├── orchestrate-planner.md    # root + subplanner contract
│   ├── orchestrate-worker.md     # worker contract
│   └── orchestrate-verifier.md   # verifier contract
├── commands/
│   └── orchestrate.md            # /orchestrate:orchestrate slash command
└── skills/
    └── orchestrate/
        ├── SKILL.md              # top-level skill entry
        ├── references/
        │   ├── planner.md        # operating manual
        │   ├── handoffs.md       # handoff schemas + synthetic failures + relay
        │   └── spawning.md       # spawn prompt contract + worktree behavior
        └── schemas/
            ├── plan.schema.json
            └── state.schema.json
```

## Quick usage

After install + restart, type `/orchestrate:orchestrate <goal>` in any Claude Code session. The session becomes the root planner. It will publish a task plan, pause for your approval, then drive the reconcile loop until terminal.

The planner does no coding — every file modification happens inside a worker task. Workers run in isolated git worktrees and communicate up via structured handoff files. See `skills/orchestrate/references/planner.md` for the load-bearing operating manual.
