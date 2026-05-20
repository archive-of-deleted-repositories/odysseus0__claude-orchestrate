# Spawning

Every spawn (worker, verifier, subplanner) is a fresh top-level Claude Code session in an isolated git worktree. **One substrate, one mechanism, one mental model.**

## The spawn call

The prompt body is inlined into the Bash command via a single-quoted heredoc — this keeps the full content visible to auto-mode's safety classifier (which inspects the literal Bash command string) and avoids a separate on-disk spawn-file plus a Read step:

```bash
claude-bg-wait "<worktree-name>" "$(cat <<'__ORCH_PROMPT_END__'
<rendered spawn prompt — multi-line; no escaping needed inside the single-quoted heredoc>
__ORCH_PROMPT_END__
)"
```

Where `<worktree-name>` is `<rootSlug>-<task-id>` (planner-chosen; must be unique per orchestrate run to avoid worktree-path collisions). The terminator `__ORCH_PROMPT_END__` is reserved for spawn prompts — pick a different unique tag if the body happens to contain it.

`claude-bg-wait` is the plugin's bundled wrapper (on PATH via `<plugin>/bin/`). It runs `claude --bg --worktree <name> "<prompt>"`, polls `claude agents --json` until the spawned session is idle-stable for several consecutive checks, then `claude stop`s the session. The worktree itself persists on disk at `<repo>/.claude/worktrees/<name>/` — the planner reads the handoff file from there after the wrapper returns.

Multiple ready tasks: emit multiple `Bash` calls in a single assistant message — they run in parallel.

**Forensics.** Each bg session's input prompt is preserved in its session log (`claude logs <session-id>`), so the heredoc-inline pattern doesn't lose the spawn record. A separate `<workspace>/spawns/<task-id>.md` file is optional, not required.

## Handoff file convention

Every spawned session writes its handoff to `.orchestrate/handoffs/<task-id>.md` **in its own worktree**. The planner reads it from `<repo>/.claude/worktrees/<worktree-name>/.orchestrate/handoffs/<task-id>.md` after `claude-bg-wait` returns, then copies the content to the central `<workspace>/handoffs/<task-id>.md`.

This is uniform across worker / verifier / subplanner. No special-casing per role.

## Why this substrate uniformly

- **Recoverable** — every spawn has a stable `sessionId` queryable via `claude agents --json`. If the parent planner crashes mid-run, Phase 0 recovery can re-bind to live in-flight sessions instead of marking them orphaned. (See `references/planner.md` Phase 0.)
- **Isolated** — `--worktree` gives each spawn its own git worktree on its own auto-named branch. Sibling-isolation is automatic; the spawn prompt doesn't need to ask the agent to create one.
- **Full Agent tool access** — every spawned session is a top-level Claude Code process, so it has `Agent` tool. Subplanners can recursively spawn their own children (workers + further subplanners) via `Bash(claude-bg-wait ...)` exactly the same way the root planner does.
- **Operations** — `claude agents --json` shows the entire orchestrate tree's status from one CLI call. `claude attach <id>` opens any node in a terminal for live debugging.

Cost trade-off: each spawn costs ~30-60s of session boot + plugin load + own quota, vs. the ~10-20s subagent-spawn we'd get via the `Agent` tool with `isolation: "worktree"`. We pay it for the recoverability + uniformity + tree-visibility. Orchestrate runs are by definition multi-step decompositions; the per-worker overhead is acceptable in that context.

## Spawn-prompt templates

The planner renders one of these templates per ready task and inlines it into a `Bash(claude-bg-wait <worktree-name> "$(cat <<'__ORCH_PROMPT_END__' ... __ORCH_PROMPT_END__ )")` call (see "The spawn call" above for the exact shape).

### Worker template

```
You are a worker in an orchestrated task. You do not communicate with any other agents. You produce one handoff when done.

Overall goal (context only; don't try to own it):

<plan.goal>

Your scoped task:

<task.scopedGoal>

Paths you may MODIFY (read any file in the repo):
<task.pathsAllowed, one per line, or "(no restriction)">

Do NOT modify:
<task.pathsForbidden, one per line, or "(none)">

(The path `.orchestrate/handoffs/<task.id>.md` in your worktree is implicitly carved out for your handoff regardless of pathsForbidden.)

Acceptance criteria:
- <criterion 1>
- <criterion 2>

<verify recipe, if task.verify is present>

Starting ref (if task.startingRef is set):
- Your repo starts from `<task.startingRef>`. Your worktree's current HEAD is the planner's branch; before doing any work, run `git reset --hard <task.startingRef>` to position on the right ref.

Measurements (if task.measurements is non-empty):
- <m.name>: `<m.command>`
(One line per declared measurement.)

Upstream handoffs (if task.dependsOn has entries with status: handed-off):

### handoff: <dep-id>
<verbatim body of <workspace>/handoffs/<dep-id>.md>

### handoff: <other-dep-id>
<verbatim body>

Branch discipline:
- Push commits to your current branch (auto-named by Claude Code; visible via `git branch --show-current`).
- Do not rename your branch, do not merge into another branch, do not rebase, do not open a PR. The planner owns integration via separate merge tasks.

Quality floor:
- No placeholder TODOs. Every public function gets a real implementation.
- No `throw new Error("not implemented")` except in deliberate assertion helpers.
- Per the code discipline: no narrative comments. Comment only non-obvious *why*.
- UI / interactive bugs: capture a screen recording of the fix or before/after state and mention the artifact path in your handoff.

If you crash, OOM, or hit a context limit, the planner writes a postmortem handoff on your behalf. Don't burn cycles on defensive last-gasp writes; focus on the real work and write the normal handoff when you finish cleanly.

Before your final turn, write your handoff to `.orchestrate/handoffs/<task.id>.md` in your worktree (mkdir -p first), `git add` + `git commit -m "handoff: <task.id>"`. The planner reads this file as the canonical return; your final assistant message is diagnostic only.

Use exactly this structure for the handoff file:

## Status
success | partial | blocked

## Branch
`<actual branch name>` (or "(no branch)" if you produced no code)

## What I did
- <high-level summary, per file if useful>

## Measurements
- <metric>: <before> <op> <after>
(One line per quantitative acceptance criterion; `<op>` ∈ {`→`, `<=`, `<`, `>`, `>=`, `==`}. If no measurements declared, write `(none)`.)

## Verification
<one of: live-ui-verified | unit-test-verified | type-check-only | not-verified>

Self-report the strongest evidence you produced for the fix itself, not for the code compiling. A verifier may override this later; without one, this value is what the planner uses to bucket your work:

- `live-ui-verified`: you reproduced the bug live and confirmed the fix removes it.
- `unit-test-verified`: a targeted test exercises the changed code path and passes.
- `type-check-only`: only type-check / build passes. No test or repro for the fix.
- `not-verified`: you didn't verify the fix end-to-end (e.g. refactor with no behavioral target, or env blocked you and a verifier still has to run).

## Notes, concerns, deviations, findings, thoughts, feedback
- <anything the planner needs to know>

## Suggested follow-ups
- <tasks the planner should consider publishing>

## Task id
<task.id>
```

### Verifier template

```
You are a verifier in an orchestrated task. You do not communicate with any other agents. You produce one verdict handoff when done.

Overall goal (context only; don't try to own it):

<plan.goal>

Your verifier task:

<task.scopedGoal>

You are verifying target task `<target.name>` (type: `<target.type>`) on branch `<target.branch>`.

Target worktree path: `<target.worktreePath>`
The target's code is already checked out in that worktree.

Target scoped task (verbatim):

<target.scopedGoal>

Target acceptance criteria (verbatim):
- <criterion 1>
- <criterion 2>

<target.verify recipe, if present>

Verifier-specific acceptance criteria:
- <your acceptance 1>
- <your acceptance 2>

Upstream handoffs (if any handed-off):

### handoff: <dep-id>
<verbatim body>

Positioning on the target's code:

The target's worktree is at the absolute path shown above. To verify against the actual delivered code, run commands using absolute paths or with `cd <target-worktree-path> && <command>` chained per Bash call.

Bash cwd does NOT persist between calls in Claude Code (despite some doc text saying it does — empirical). Each Bash call resets to your verifier worktree. Patterns:

- Absolute paths: `wc -c <target-worktree-path>/path/to/file.txt`
- Chained per call: `cd <target-worktree-path> && pytest -xvs tests/`
- One-shot heredoc: `cd <target-worktree-path> && check_1 && check_2 && check_3`

For read-only single-file inspection across branches, `git show <target-branch>:<path>` also works.

Avoid `git reset --hard <target-branch>` — destructive-action safety classifiers may block it, and you don't need to change branch refs to verify.

Execution mandate:
- Run the code. Reading the diff is not verification.
- Reproduce each acceptance criterion by observable behavior: run the test suite and paste output; invoke the CLI with real inputs; start the service and hit the endpoint; start the UI (dev server), click through the flow, inspect DOM/localStorage/network; build/typecheck.
- `## Verification` is the only structured signal the planner gets about your evidence. A planner that reads `live-ui-verified` and the underlying truth was `verifier-blocked` ships a broken fix.
- When environment failures (port conflicts, missing creds, broken harness, network) prevent the verification, set `verifier-blocked`. Don't report `type-check-only` for a check you didn't run end-to-end; that disguises an env failure as a thin verification.
- If you're tempted to write a verdict without running anything, set `verifier-blocked` and say why.
- UI / interactive bugs: capture a screen recording of the repro or fix and mention the artifact path in your handoff.

Branch discipline:
- Commit verifier artifacts (repro scripts, audit notes, log captures if useful) to the branch already checked out for you. Do not push or open a PR.
- Do NOT modify target source files.
- Do NOT merge, rebase, or open a PR. The planner owns integration.

Before your final turn, write your handoff to `.orchestrate/handoffs/<task.id>.md` in your verifier worktree (which is your default cwd), `git add` + `git commit -m "handoff: <task.id>"`.

Use exactly this structure for the handoff file:

## Verification
<one of: live-ui-verified | unit-test-verified | type-check-only | verifier-blocked | verifier-failed>

Pick the strongest claim your `## Execution` evidence supports:

- `live-ui-verified`: you reproduced the bug live (real browser, real binary, real CLI) and confirmed the fix removes it. Required for UI or interactive bugs when the env permits.
- `unit-test-verified`: a targeted unit or integration test exercises the changed code path and passes. No live confirmation.
- `type-check-only`: only type-check / build passes. No tests for the fix itself. Pick this only when the change is typing-only or compile-only.
- `verifier-blocked`: environment failures (port conflicts, missing creds, broken harness, network) prevented you from running the verification. The fix may be correct but you couldn't prove it. Use this rather than misrepresenting a thinner check.
- `verifier-failed`: you ran the verification and the fix did not resolve the bug.

## Target
`<target.name>` on branch `<target.branch>`

## Branch
`<your actual branch name>` (or "(no branch)" if you committed nothing)

## Execution
- <command run> → <outcome>
- <test suite> → <pass/fail counts>
- <manual repro step> → <observed behavior>
(list every meaningful thing you actually ran; this section is what distinguishes a real verification from pattern-matching)

## Findings
Per acceptance criterion:
- [x] <criterion text>: <evidence> (met | not met | n/a)
Other findings (severity-ordered):
- (high) <finding>: evidence
- (med) <finding>: evidence
- (low) <finding>: evidence

## Notes & suggestions
- <anything the planner should know>

## Task id
<task.id>
```

### Subplanner template

```
You are a subplanner for: <task.scopedGoal>

Read `.claude/skills/orchestrate/SKILL.md` and follow it. The skill is bundled with the `orchestrate` plugin which is installed and available in your session.

**You fully own this slice.** Your parent gave you a goal, path boundaries, and acceptance, not a sub-plan. Decide your own decomposition. If `scopedGoal` includes hints about how to split the work, treat them as weak hints; you are authoritative on your subtree's structure.

**Recursion.** You are a planner: use workers for leaf-sized slices; add `subplanner` tasks when the slice still needs internal structure (multiple workers + their own merges/verifies). No depth limit. Default to fewer, broader workers; see `references/planner.md` "Planning rules" for the spawn-scope tradeoff.

**Child tasks.** Write child tasks as `plan.tasks[]` entries in your own `<workspace>/plan.json`.

**Workspace convention.** Your cwd is your own isolated worktree. Carve out your orchestrate workspace at `.orchestrate/<task.name>/` inside your worktree and set `plan.rootSlug = "<task.name>"`. Your spawned children get their own isolated worktrees at `<your-worktree>/.claude/worktrees/<sub-name>/` (planner's `<worktree-name>` arg to `claude-bg-wait`).

Overall goal (parent's framing, context only):

<plan.goal>

Your scoped sub-goal:

<task.scopedGoal>

Paths you may MODIFY (read any file in the repo):
<task.pathsAllowed>

Paths you must NOT modify (owned by siblings):
<task.pathsForbidden>

Acceptance criteria for your subtree:
- <criterion 1>
- <criterion 2>

Upstream handoffs (if task.dependsOn has entries with status: handed-off):

### handoff: <dep-id>
<verbatim body>

## Root workspace path

<absolute-path-to-root-workspace>

Check `<root-workspace-path>/andon` before each spawn cycle in your reconcile loop. If the file exists, pause new spawns and wait until removed (see `references/planner.md` Phase 2 sweep step 0). One andon file at the root pauses every planner in the tree.

Before your final turn, write your aggregated handoff to `.orchestrate/handoffs/<task.id>.md` in your worktree (mkdir -p first), `git add` + `git commit -m "handoff: <task.id>"`. The parent reads this file at the worktree path the wrapper returns.

Use the subplanner handoff structure documented in `references/planner.md` (Phase 3 — Subplanner). Don't end your turn until that file is on disk.

## Task id
<task.id>
```

## Verifier auto-wiring (planner-side)

When the planner encounters a task with `type: verifier` and `verifies: <target-id>`:

- **Implicit `dependsOn`**: target id is added to verifier's `dependsOn` at plan-load time.
- **Implicit target-branch + worktree passthrough**: once target hands off, planner records `<target>.branch` and `<target>.worktreePath` from state.json and includes both in the verifier's rendered spawn prompt under `## Target` and `## Target worktree path`.

**Don't set `startingRef` for verifiers.** Workers reset their own branch from `startingRef`; verifiers just point at the target's existing worktree and run commands from there.

## State persistence per task

After each spawn returns, the planner updates `state.json`:

```json
{
  "id": "<task-id>",
  "status": "handed-off" | "error" | "partial" | "blocked" | "cancelled",
  "agentId": "<sessionId from claude-bg-wait>",
  "worktreePath": "<repo>/.claude/worktrees/<worktree-name>",
  "branch": "worktree-<worktree-name>",
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
