---
name: jira-ticket-creator
description: >
  MANDATORY TRIGGERS: "start jira-ticket-creator", "create a jira ticket",
  "log a bug", "open a task", "new story", "create epic", "file a ticket",
  "new jira issue". Starts an interactive JIRA ticket creation flow in the
  ACD (Aqua Claude Dev) space. Do NOT scan or read any other project files —
  go straight to Step 1 of the flow.
---

# Jira Ticket Creator

Walk the user through creating a Jira ticket one question at a time. Ask each question separately — never combine steps. Wait for the user's response before moving to the next step.

**Shared Protocols:** This skill references reusable protocols in `.claude/skills/jira/_shared/references/`. Read the relevant protocol file when a step says "follow [protocol-name]".

## Section IDs

**Prefix:** `tc`

When starting each step, display the section ID:

```
▶ [tc.N.name] — Step title
```

| ID | Step |
|---|---|
| `tc.1.issue-type` | Ask issue type |
| `tc.2.summary` | Ask for summary |
| `tc.3.description` | Ask for description |
| `tc.4.acceptance-criteria` | Acceptance criteria checklist |
| `tc.5.priority` | Ask for priority |
| `tc.6.review-confirm` | Review and confirm |
| `tc.7.create-ticket` | Create the JIRA ticket |

## Flow

### Step 1 — Ask issue type {#tc.1.issue-type}

Use `AskUserQuestion` with these 4 options:

- **Task** — A unit of work (development, research, config, etc.)
- **Bug** — Something is broken or behaving unexpectedly
- **Epic** — A large body of work spanning multiple stories/tasks
- **Story** — A user-facing feature from the user's perspective

Wait for the user's selection before continuing.

### Step 2 — Ask for summary {#tc.2.summary}

Do NOT use AskUserQuestion here. Instead, reply with a plain text message asking:

> "What's the ticket summary? (A short title, 5–15 words)"

Wait for the user to type their answer as a normal chat message.

### Step 3 — Ask for description {#tc.3.description}

Do NOT use AskUserQuestion here. Instead, reply with a plain text message. Tailor the question based on the issue type they picked in Step 1:

- If **Task**: "Describe what needs to be done."
- If **Bug**: "Describe the bug — steps to reproduce, expected behavior, and actual behavior."
- If **Epic**: "Describe the goal, scope, and success criteria for this epic."
- If **Story**: "Describe the story as: As a [who], I want [what], so that [why]."

Wait for the user to type their answer as a normal chat message.

### Step 4 — Acceptance criteria {#tc.4.acceptance-criteria}

**Follow the Acceptance Criteria Checklist Protocol** in `.claude/skills/jira/_shared/references/acceptance-criteria-checklist.md`.

Based on the issue type, summary, and description collected so far, generate a set of suggested acceptance criteria tailored to this specific ticket. Present them as a numbered checklist:

```
Suggested Acceptance Criteria:

Based on your description, here are the criteria I'd recommend:

  1. ✅ [criterion 1 — specific to this ticket]
  2. ✅ [criterion 2 — specific to this ticket]
  3. ✅ [criterion 3 — specific to this ticket]
  4. ✅ [criterion 4 — specific to this ticket]

All are selected by default.
```

Use `AskUserQuestion`:

- **Keep all, looks good** — Use all suggested criteria as-is
- **Let me select which ones to keep** — I want to remove some
- **Keep all and add more** — These are good, plus I have additional ones
- **Start fresh** — I want to write my own from scratch

If selecting: ask which numbers to remove, re-present, then confirm.
If adding more: collect additional criteria, add to list, re-present, then confirm.
If starting fresh: collect user's criteria, format as bullets, confirm.

Repeat until the user confirms the final acceptance criteria list.

### Step 5 — Ask for priority {#tc.5.priority}

Use `AskUserQuestion` with these 4 options:

- **Highest** — System down, data loss, no workaround
- **High** — Major feature broken, workaround exists but painful
- **Medium** — Important but not urgent
- **Low** — Nice to have, minor improvement

### Step 6 — Review and confirm {#tc.6.review-confirm}

Rephrase the user's summary and description for clarity and professionalism. Present the SUGGESTED version and explicitly invite modifications:

```
Here's how I'd phrase your ticket. Review and let me know if you'd like to adjust anything:

Type:        [selected type]
Summary:     [rephrased summary — suggested improvement]
Priority:    [selected priority]

Description:
[rephrased description — suggested improvement]

Acceptance Criteria:
- [criterion 1]
- [criterion 2]
- ...
```

Then use `AskUserQuestion`:

- **Yes, create it** — Looks good, create the ticket in JIRA
- **No, let me edit** — I want to change something
- **Looks good but I want to add more details** — I have additional context to include

If the user selects **"No, let me edit"** or **"add more details"**, ask which field they want to change (type, summary, description, acceptance criteria, or priority), collect the new value, then show the review again. Repeat until they confirm.

### Step 7 — Create the JIRA ticket {#tc.7.create-ticket}

Only after the user confirms, create the ticket directly in JIRA using the `createJiraIssue` MCP tool with the following details:

- **Project key:** `ACD` (Aqua Claude Dev Space)
- **Issue type:** as selected by the user
- **Summary:** rephrased summary (as confirmed by user)
- **Description:** rephrased description, followed by the acceptance criteria formatted as bullet points under an "Acceptance Criteria" heading
- **Priority:** as selected by the user

After successful creation, display the ticket key and link:

```
Ticket Created: [KEY]
Link: https://vinay251101.atlassian.net/browse/[KEY]

Summary: [summary]
Type: [type] | Priority: [priority]
```

Use `AskUserQuestion`:

- **Done** — That's all for now
- **Create another ticket** — I want to create another one
