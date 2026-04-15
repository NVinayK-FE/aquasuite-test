---
name: jira-router
description: >
  Use this skill whenever a message mentions a JIRA ticket key (e.g., ACD-18)
  with intent to implement or execute it — phrases like "implement <TICKET-ID>",
  "execute <TICKET-ID>", "work on <TICKET-ID>", "pick up <TICKET-ID>",
  "start <TICKET-ID>", "<TICKET-ID> give steps", or even a bare ticket ID on
  its own. The router fetches the ticket, detects its type (Epic, Story, Task,
  Subtask, Bug), and dispatches to the correct executor so the user never has
  to remember which skill to call. Prefer routing through this skill over
  guessing the executor directly.
---

# JIRA Router

Automatically detect a JIRA ticket's type and route to the correct executor skill. The user only needs to provide a ticket ID — no need to remember specific trigger commands.

## Flow

### Step 1 — Fetch the ticket

Use the `getJiraIssue` MCP tool to fetch the ticket by its key. Extract the `issuetype.name` field.

### Step 2 — Route by type

Based on the issue type, determine the correct skill and present the routing to the user:

| Issue Type          | Route To                                          | Command Equivalent                        |
|---------------------|---------------------------------------------------|-------------------------------------------|
| **Epic**            | `jira-epic-orchestrator` → execute mode            | `execute epic <TICKET-ID>`                |
| **Story**           | `jira-story-executor`                              | `execute story <TICKET-ID>`               |
| **Task**            | `jira-executor`                                    | `start jira-executor <TICKET-ID>`         |
| **Subtask**         | `jira-executor`                                    | `start jira-executor <TICKET-ID>`         |
| **Bug**             | `jira-bug-executor`                                | `start jira-bug-executor <TICKET-ID>`     |

Present to the user:

```
🔍 Ticket:  [KEY] — [Summary]
📋 Type:    [Issue Type]
🔀 Routing: → [Skill Name]
💬 Command: [equivalent trigger command]
```

### Step 3 — Confirm before handoff

After presenting the routing info, use `AskUserQuestion` to confirm before proceeding:

- **Yes, proceed** — Hand off to the detected skill and begin execution
- **Just analysing** — Show ticket details only, do not start any executor flow
- **Wrong routing, let me pick** — I want to choose a different skill manually

If the user selects **"Yes, proceed"**, continue to Step 4.

If the user selects **"Just analysing"**, present a detailed analysis of the ticket (summary, description, acceptance criteria, status, priority, parent, assignee) and stop. Do NOT start any executor flow.

If the user selects **"Wrong routing, let me pick"**, present the full list of available skills and let the user choose which one to use.

### Step 4 — Hand off

Read the target skill's `SKILL.md` and begin executing its flow from Step 1 — passing along the ticket ID. Do NOT ask the user to re-type a command. The handoff must be seamless.

**Important:** Once the user confirms, the target skill takes over completely. The router's job is done — it only saves the user from having to know which command to use.
