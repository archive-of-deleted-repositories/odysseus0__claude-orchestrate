# Changelog

All notable changes to `claude-orchestrate` are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.0] - 2026-05-20

Initial public release. Adapted from Cursor's [`/orchestrate`](https://github.com/cursor/plugins/tree/main/orchestrate) plugin. Same five-primitive design (Task / Handoff / Workspace / Roles / Loop), reimplemented for Claude Code's substrate: every spawned task is a fresh top-level Claude Code session, role contracts live in prompt templates rather than agent definitions, handoffs flow through files in isolated git worktrees, and a daemon-supervised background runtime replaces Cursor's in-process subagent calls.

### Added

- **Uniform `claude --bg` spawn substrate.** Every task type — worker, verifier, and recursive subplanner — is spawned through the same `claude-bg-wait <worktree-name> <prompt>` wrapper. One mechanism, one mental model: no special case for "this kind of agent gets called this way." The wrapper handles worktree creation, background session launch, completion notification, and exit-code propagation uniformly across all roles.

- **Recursive subplanners via `claude --bg --worktree`.** Subplanners are not subagents in the SDK sense — they are full top-level Claude Code sessions launched in their own worktrees, each with unrestricted Agent-tool access. This means a subplanner can itself decompose its scoped goal into further worker/verifier/subplanner trees, with no fan-out ceiling baked into the substrate. The recursion is bounded by the planner's own judgement, not by tool-permission scaffolding.

- **File-canonical handoffs.** Every spawned session writes its result to `.orchestrate/handoffs/<task-id>.md` inside its own worktree before exiting. The parent planner reads from there. No structured-return-value contract between agents — the handoff file is the interface. This makes spawned tasks fully auditable (every handoff is a committed artifact in its worktree branch) and lets planners reconcile asynchronously without holding child-session output in context.

- **Substrate-clean role contracts.** No `agents/` directory and no Claude Code subagent definitions. Worker, verifier, and subplanner role contracts live as prompt templates in `skills/orchestrate/references/spawning.md` and are injected into the spawn prompt at dispatch time. This matches Cursor's prompt-canonical design — the role *is* the prompt, not a separate config file that has to stay in sync with one.

- **File-based Andon (cross-tree pause).** Operator touches `.orchestrate/<rootSlug>/andon` from any terminal to halt all new spawns across the orchestration tree. Every planner — root and every subplanner — checks the root's andon flag file before each spawn cycle, regardless of which worktree the planner itself is running in. Provides a single cross-tree kill switch for runaway fan-out, expensive runs, or any "stop and let me look" moment, without requiring the operator to know which session is which.

- **Atomic state writes via `orchestrate-state-write`.** The helper writes orchestration state through a tempfile + atomic rename rather than a direct overwrite, so a mid-write crash, SIGKILL, or disk-full event leaves the previous state intact instead of producing a half-written, unparseable file. Critical because the orchestrate loop reads state on every reconcile cycle — partial-file corruption on a long-running tree would otherwise force a manual recovery.

- **Verifier auto-wiring (cd-into-target).** Verifiers position themselves at the target worktree by `cd`-ing into the target's existing worktree path, instead of doing a `git reset --hard <target-branch>` (which the auto-mode classifier blocks as destructive) or a `git checkout <target-branch> -- .` (which clobbers untracked target files). The planner includes the resolved worktree path under a `## Target worktree path` heading in the verifier's spawn prompt; the verifier prompt template instructs the agent to `cd` there on first action and keep absolute paths in subsequent Bash calls (the cwd does not persist between Bash invocations).

- **Plugin packaging with `bin/` scripts on PATH.** Claude Code auto-prepends each installed plugin's `bin/` directory to `PATH` for every session it spawns. `claude-bg-wait` and `orchestrate-state-write` are shipped under `bin/` so they are callable as bare command names from any orchestrate-spawned session — no path resolution, no `$CLAUDE_PLUGIN_ROOT` indirection at the call site. Plugin packaging metadata (`plugin.json`, `marketplace.json`) was trimmed to match the format used by sibling plugins (`codex`, `obsidian-skills`).

### Notes

- Verifier prompts retain Cursor's stricter Verification self-report wording — the verifier must state exactly what it ran and what it observed, not a paraphrase. Preserved verbatim from the upstream design rather than softened during the port.
- The previous `orchestrate-planner` Claude Code subagent has been removed. Planning is the loop itself, running in whatever session invoked `/orchestrate`; worker and verifier prompts are now injected verbatim from the spawning reference rather than indirected through a subagent definition.
