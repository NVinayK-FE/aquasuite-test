# Executor Comparison

Quick reference for when to pick which JIRA executor skill.

**Single execution path.** As of the current project rules, **all per-ticket work ultimately runs through `jira-executor`**. The other executor skills are now **thin orchestrators** that load context, analyse, or iterate a queue, and then hand each ticket off to `jira-executor` for the full understand → dev-flows → spec-first → plan → implement → transition → commit → jira-update flow. The table below reflects that model.

## At a glance

| Skill | Input | Scope | What it does itself | Ultimately executes per-ticket via |
|---|---|---|---|---|
| `jira-router` | 1 ticket key | Any single ticket | Detect type, confirm routing, hand off | `jira-executor` (possibly via one of the skills below) |
| `jira-executor` | 1 ticket key | Single Task or Subtask | Full per-ticket flow (understand → dev-flows → spec-first → plan → implement → transition → commit → jira-update) | `jira-executor` (itself) |
| `jira-bug-executor` | 1 Bug ticket key | Single Bug | Reproduce, root-cause, confirm diagnosis, write analysis block to ticket description, hand off | `jira-executor` |
| `jira-story-executor` | 1 Story key | Story + all its subtasks | Load story context, transition story to In Progress on first subtask, iterate subtasks with per-subtask issue-type re-detection, wrap up with an aggregated story-level comment | `jira-executor` (per subtask) — Bug subtasks detour via `jira-bug-executor` first |
| `jira-batch-executor` | Comma-separated keys | Multiple independent tickets (same or different projects) | Iterate queue, warn on cross-project batches, hand off | `jira-executor` (per ticket) — Bug tickets detour via `jira-bug-executor` first |
| `jira-epic-orchestrator` (execute mode) | Epic key | Epic + all its children (stories/tasks/bugs) | Load epic, transition epic to In Progress on first child, iterate children, wrap up with aggregated epic-level comment | `jira-executor` (per terminal ticket) — Stories go through `jira-story-executor`, Bugs through `jira-bug-executor` |

## Key distinctions

**`jira-executor` is the only code-writing skill.** Every other executor in the table above is an orchestrator or analyser. They may edit the ticket (bug description, story-level comment, epic-level comment) but they never edit source code, they never run the spec-first check, they never produce `code-to-phrase` / `code-to-sentence`, they never `git commit`, and they never post per-ticket implementation comments. That discipline keeps the execution path single and prevents drift between skills.

**`jira-story-executor` vs `jira-batch-executor`.** Batch treats each ticket as independent — issue type is re-detected per ticket, and each ticket gets its own full spec-first run. Story treats the story as a cohesive unit — the parent story's specs are shared context across its subtasks, but each subtask still runs its own per-subtask spec-first check inside `jira-executor` (because specs can be touched by any subtask). Use batch for unrelated work, story for cohesive work under one parent.

**`jira-story-executor` vs `jira-epic-orchestrator` execute.** The epic orchestrator drives an entire epic, which typically contains multiple stories and/or tasks. It delegates Stories to `jira-story-executor` (which in turn delegates subtasks to `jira-executor`) and Tasks/Bugs directly to `jira-executor` / `jira-bug-executor`. Use story when you want to finish one story; use epic when you want to drive several stories/tasks in a single session.

**`jira-executor` vs `jira-bug-executor`.** For Bug tickets, always enter through `jira-bug-executor` first. Its analysis step (reproduce → root-cause → confirm → write analysis block to description) cannot be replicated inside `jira-executor` and must happen before any fix is written. Once the analysis block is on the ticket, `jira-executor`'s Step 1 has a fast-path that detects the block and auto-confirms the understanding, so the user is not re-asked.

## Git control

All git work happens inside `jira-executor` as a single integrated step (Step 8 in that skill), structured as separate commits for specs → dev-flows → code. The higher-level orchestrators (story, batch, epic) do **not** run their own git flow — they rely on `jira-executor`'s per-ticket commit discipline. There is no longer a "Level 1 / Level 2 / Level 3" tiering; there is only `jira-executor`'s flow, and the orchestrators let it run per ticket.

See `jira-executor/SKILL.md` Step 8 for the actual commit / push / branch handling.

## Remember-answer protocol

Every orchestrator must follow [`hcp.8.remember-answer`](./human-confirmation-protocol.md). The between-ticket "Continue / Pause" prompts in the story, batch, and epic skills all offer "Always continue this session" as a remember variant so the user can set-and-forget a long queue. Destructive prompts (status → Done transitions, git push) never carry a remember variant — see [`hcp.6.destructive-pattern`](./human-confirmation-protocol.md).
