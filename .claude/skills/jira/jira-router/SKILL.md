---
name: jira-router
description: >
  Use this skill whenever a message mentions a JIRA ticket key (e.g., ACD-18)
  with intent to implement or execute it — phrases like "implement <TICKET-ID>",
  "execute <TICKET-ID>", "work on <TICKET-ID>", "pick up <TICKET-ID>",
  "start <TICKET-ID>", "<TICKET-ID> give steps", or even a bare ticket ID on
  its own. The router fetches the ticket, detects its type (Epic, Story, Task,
  Subtask, Bug), and dispatches the work so the user never has to remember
  which skill to call. All actual ticket execution ultimately runs through
  `jira-executor`; other skills either plan upstream or produce context that
  feeds into it.
---

# JIRA Router

Automatically detect a JIRA ticket's type and route to the correct upstream skill. **All implementation ultimately runs through `jira-executor`** — the orchestrators (epic, story, batch) and `jira-bug-executor` handle planning, context loading, or analysis, and then hand off per-ticket execution to `jira-executor`.

## Per-skill rule — Remember answers for the session {#router.remember}

Every `AskUserQuestion` prompt raised by this skill MUST follow the Human Confirmation Protocol's **Remember Answer for the Session** rule (see `.claude/skills/jira/_shared/references/human-confirmation-protocol.md` — section [`hcp.8.remember-answer`]).

Concretely:

- The Step 3 confirm-before-handoff prompt is asked per-ticket. If the user has selected "Yes, proceed" for several consecutive tickets in this session, offer a remember variant ("Always accept detected routing this session") so the router becomes invisible for trusted, well-typed tickets.
- When a remembered choice is in effect, print `▸ Using remembered routing choice: auto-accept` and proceed straight to handoff.
- The "Wrong routing, let me pick" branch is an override — it never triggers a remember variant, and any previously remembered auto-accept is cleared when the user takes this branch (so a wrong detection doesn't quietly keep auto-accepting).
- Downstream skills keep their own per-skill `hcp.8` handling.

## Principle — Single Execution Path {#router.principle}

The actual "understand → dev flows → specs → plan → implement → transition → commit → jira update" workflow lives in **`jira-executor` only**. No other skill reimplements those steps. Upstream skills do their upstream work and then delegate.

- **Epic** → plan/orchestrate in `jira-epic-orchestrator`; for each pending story/task, delegate to `jira-executor` (directly for a Task, or via `jira-story-executor` for a Story with subtasks).
- **Story** → load story + subtasks in `jira-story-executor`; for each pending subtask, delegate to `jira-executor`.
- **Task / Subtask** → route straight to `jira-executor`.
- **Bug** → run the analysis phase in `jira-bug-executor`; once the user confirms the diagnosis, update the ticket description with the analysis and hand off to `jira-executor` for the actual fix.
- **Multiple standalone tickets** → queue in `jira-batch-executor`; for each ticket, delegate to `jira-executor` (routing through `jira-bug-executor` first if the ticket is a Bug).

## Flow

### Step 1 — Fetch the ticket

Use the `getJiraIssue` MCP tool to fetch the ticket by its key. Extract the `issuetype.name` field.

### Step 2 — Route by type

Based on the issue type, determine the routing and present it to the user:

| Issue Type  | Upstream skill                                                            | Ultimately executes via | Command Equivalent                    |
|-------------|---------------------------------------------------------------------------|-------------------------|---------------------------------------|
| **Epic**    | `jira-epic-orchestrator` → execute mode                                    | `jira-executor` (per child ticket) | `execute epic <TICKET-ID>`   |
| **Story**   | `jira-story-executor` (load context + iterate subtasks)                    | `jira-executor` (per subtask)      | `execute story <TICKET-ID>`  |
| **Task**    | `jira-executor` (direct)                                                   | `jira-executor`         | `start jira-executor <TICKET-ID>`     |
| **Subtask** | `jira-executor` (direct)                                                   | `jira-executor`         | `start jira-executor <TICKET-ID>`     |
| **Bug**     | `jira-bug-executor` (analysis → confirm → update description → handoff)    | `jira-executor`         | `start jira-bug-executor <TICKET-ID>` |

Present to the user:

```
🔍 Ticket:    [KEY] — [Summary]
📋 Type:      [Issue Type]
🔀 Upstream:  → [Upstream Skill]
⚙️  Executes:  → jira-executor (per ticket)
💬 Command:   [equivalent trigger command]
```

### Step 3 — Confirm before handoff

After presenting the routing info, use `AskUserQuestion` to confirm before proceeding:

- **Yes, proceed** — Hand off to the detected upstream skill and begin execution
- **Just analysing** — Show ticket details only, do not start any executor flow
- **Wrong routing, let me pick** — I want to choose a different skill manually

If the user selects **"Yes, proceed"**, continue to Step 4.

If the user selects **"Just analysing"**, present a detailed analysis of the ticket (summary, description, acceptance criteria, status, priority, parent, assignee) and stop. Do NOT start any executor flow.

If the user selects **"Wrong routing, let me pick"**, present the full list of available skills and let the user choose which one to use.

**Wrong-routing guard.** Before handing off to a user-picked skill, check that the skill's expected input shape matches the ticket actually fetched in Step 1. Specifically:

| User-picked skill          | Expected ticket shape                                                     |
|----------------------------|---------------------------------------------------------------------------|
| `jira-executor`            | Type is Task or Subtask (not Epic, not Story with subtasks, not Bug)      |
| `jira-bug-executor`        | Type is Bug                                                               |
| `jira-story-executor`      | Type is Story AND at least one subtask exists                             |
| `jira-epic-orchestrator`   | Type is Epic                                                              |
| `jira-batch-executor`      | Multiple ticket keys were provided (not a single-key router invocation)   |

If the user's pick does not match the fetched ticket's shape (e.g., they chose `jira-story-executor` for a Bug ticket, or `jira-executor` for an Epic), print a warning and re-confirm before handoff:

```
⚠ Manual routing mismatch
  Ticket:         [KEY] — [Summary]
  Detected type:  [Epic | Story | Task | Subtask | Bug]
  You picked:     [skill]
  Expected shape: [expected shape for that skill]

The picked skill may fail fast or produce the wrong flow for this ticket.
```

Then offer:

- **Proceed anyway** — I know what I'm doing; hand off as picked
- **Take the auto-detected route instead** — Use the router's default pick
- **Cancel** — Don't route; I'll pick again

Only hand off once the user has explicitly accepted the mismatch or reverted to the default.

### Step 4 — Hand off

Read the upstream skill's `SKILL.md` and begin executing its flow from Step 1 — passing along the ticket ID. The upstream skill is responsible for delegating to `jira-executor` at the implementation boundary; the router does not call `jira-executor` directly except for Task/Subtask where the upstream *is* `jira-executor`.

Do NOT ask the user to re-type a command. The handoff must be seamless.

**Important:** Once the user confirms, the upstream skill takes over completely. The router's job is done — it only saves the user from having to know which command to use, and it enforces the "execution → jira-executor" rule by choosing the right upstream.
