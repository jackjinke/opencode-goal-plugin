# Goal Mode Improvement Plan

This plan implements the improvement list produced from comparing this plugin with OpenAI Codex native goal mode, latest OpenCode plugin/TUI APIs, and Willy Topete's `willytop8/OpenCode-goal-plugin`.

Credit: Willy's plugin directly inspired the history/checkpoint model, no-progress pause heuristic, budget wrap-up behavior, owner-only persistence hardening, and safer idempotent system prompt injection strategy. This implementation keeps this package's Codex-style tool-based completion model while adapting those ideas to the existing TypeScript state API and OpenCode plugin hooks.

## Goals

- Keep `/goal` available across TUI, desktop, and web through the server command template.
- Preserve the public Promise-based state API while extending snapshots with budget, history, and checkpoint metadata.
- Avoid strict-provider failures by merging goal context into the primary system block instead of pushing extra system messages.
- Use OpenCode's compaction auto-continue hook to avoid generic continuation racing against goal-specific continuation.
- Add Codex-inspired statuses for safety stops: `budgetLimited` and `usageLimited`, while keeping existing `complete` and `unmet` closure semantics.
- Add per-goal token budgets, max auto-turn budgets, optional max elapsed time budgets, and configurable no-progress detection.
- Add lifecycle history and recent checkpoints so users can inspect what happened during long-running goals.
- Improve the TUI command palette entry with pause, resume, history, and clear actions while preserving the existing non-slash `Goal` command.
- Update tests and documentation, then run local regression gates before pushing.

## Implementation Steps

1. State model and persistence
   - Extend `GoalStatus` with `usageLimited`.
   - Store `tokenBudget`, `maxAutoTurns`, `maxDurationSeconds`, `noProgressTurns`, `history`, `checkpoints`, `lastCheckpoint`, `lastAssistantText`, and `lastAssistantMessageID`.
   - Decode older persisted state by adding defaults for new fields.
   - Keep atomic writes and add owner-only file permissions where supported.
   - Do not overwrite corrupt state during startup reads.

2. Goal lifecycle and budgets
   - Allow `createGoal` and `set_goal`/`create_goal` tools to accept optional `token_budget`, `max_auto_turns`, and `max_duration_seconds`.
   - Set `remainingTokens` correctly in snapshots.
   - Mark goals `budgetLimited` when token budget is exhausted and `usageLimited` when auto-turn or duration budgets are exhausted.
   - Add `updateGoalObjective` so active or paused goals can be edited explicitly.
   - Keep `complete` requiring evidence and `unmet` requiring a blocker.

3. Continuation and progress detection
   - Add checkpoint recording from assistant output when available.
   - Track repeated low-output or repeated assistant text and pause after the configured no-progress threshold.
   - Send a final budget wrap-up prompt when a budget/usage stop occurs instead of silently stopping.
   - Keep the current overlapping-continuation guard.

4. Prompt/system integration
   - Escape user-provided objectives in XML-like prompt blocks.
   - Expand continuation prompts with Codex-style fidelity, completion audit, blocked audit, and budget context.
   - Merge active goal reminders into `output.system[0]` idempotently, inspired by Willy's strict-template-provider hardening.
   - Avoid adding a no-goal system reminder on every turn.

5. OpenCode latest hook integration
   - Implement `experimental.compaction.autocontinue` to disable generic post-compaction continuation when an active goal should drive its own continuation.
   - Keep `experimental.session.compacting` context injection.

6. TUI improvements
   - Parse new snapshot fields.
   - Display token budget/remaining tokens, stop reasons, and latest checkpoint.
   - Add command palette actions for refresh, pause, resume, history, and clear.
   - Prefer modern keymap registration when available while falling back to legacy `api.command` for current compatibility.

7. Documentation and tests
   - Update README options, workflow, state, and credit sections.
   - Add or update state/server/TUI tests for budget limits, system merge idempotency, compaction autocontinue, history/checkpoints, no-progress pause, and TUI actions.
   - Build `dist/server.js` through the existing build script.

8. Release workflow
   - Run `bun run lint`, `bun run typecheck`, `bun test`, `bun run build`, and `bun run pack:dry-run`.
   - Commit only intended files, force-adding this ignored plan file.
   - Push to `main` to trigger publish/deploy.
   - Monitor the GitHub Actions publish workflow and run a post-push smoke/regression test against the published package when available.

## Non-Goals

- Do not replace OpenCode's plugin architecture or fork OpenCode.
- Do not remove existing `/goal` command template behavior.
- Do not require marker-based `[goal:complete]` completion; this plugin keeps explicit `update_goal` evidence/blocker tools.
- Do not include unrelated local files such as the pre-existing untracked `CONTEXT.md` unless separately requested.
