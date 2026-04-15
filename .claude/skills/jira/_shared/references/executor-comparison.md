# Executor Comparison

Quick reference for when to pick which JIRA executor skill. Each executor specialises for a different granularity of work — picking the right one up front avoids mid-flight skill-switching.

## At a glance

| Skill | Input | Scope | Spec-first runs | Git control |
|---|---|---|---|---|
| `jira-executor` | 1 ticket key | Single Task or Subtask | Once per ticket | Level 1 (basic) |
| `jira-bug-executor` | 1 ticket key | Single Bug | Root-cause first, then fix | Level 1 (basic) |
| `jira-story-executor` | 1 Story key | Story + all its subtasks | Once per story | Level 3 (full) |
| `jira-batch-executor` | Comma-separated keys | Multiple independent tickets | Once per ticket | Level 3 (full) |
| `jira-epic-orchestrator` (execute) | Epic key | Epic → stories → tasks | Once per story | Level 2 (standard) |

## Key distinctions

**story-executor vs batch-executor.** The batch executor treats each ticket independently and re-runs spec-first per ticket. The story executor treats the story as one unit — spec-first runs once covering all subtasks, then subtasks execute in sequence without repeating the spec check. Use batch for unrelated work, story for cohesive work under one parent.

**story-executor vs epic-orchestrator execute.** The epic orchestrator processes an entire epic (multiple stories). The story executor focuses on a single story and offers Level 3 git control (full interactive menu) rather than Level 2. Use story when you want deep control over one story; use epic when you want to drive several stories in a single session.

**executor vs bug-executor.** Regular tickets benefit from implementation-first; bugs benefit from investigation-first (reproduce → root-cause → confirm → fix). The router will normally pick the right one, but if you're choosing manually, route Bug types to bug-executor so the root-cause step isn't skipped.

## Git control levels

- **Level 1 (basic)** — commit + optional push after the ticket.
- **Level 2 (standard)** — commit, push, branch management between stories.
- **Level 3 (full)** — interactive menu covering commit, push, branch create/switch, merge, pull, PR; loops until the user exits.

See `git-workflow.md` for full details of each level.
