---
name: jira-markdown-creator
description: >
  MANDATORY TRIGGERS: "start jira-markdown-creator", "create a markdown ticket",
  "log a doc task", "new docs ticket", "create MD ticket", "new documentation issue".
  Starts an interactive JIRA ticket creation flow in the MD (Aqua Claude Docs) space.
  Do NOT scan or read any other project files — go straight to Step 1 of the flow.
---

# Jira Markdown Creator

Walk the user through creating a Jira documentation ticket in the **Aqua Claude Docs (MD)** space one question at a time. Ask each question separately — never combine steps. Wait for the user's response before moving to the next step.

All markdown/documentation JIRA tickets are created directly without requiring an existing JIRA ticket.

## Flow

### Step 1 — Ask issue type

Use `AskUserQuestion` with these 4 options:

- **Task** — A unit of work (development, research, config, etc.)
- **Bug** — Something is broken or behaving unexpectedly
- **Epic** — A large body of work spanning multiple stories/tasks
- **Story** — A user-facing feature from the user's perspective

Wait for the user's selection before continuing.

### Step 2 — Ask for summary

Do NOT use AskUserQuestion here. Instead, reply with a plain text message asking:

> "What's the ticket summary? (A short title, 5–15 words)"

Wait for the user to type their answer as a normal chat message.

### Step 3 — Ask for description

Do NOT use AskUserQuestion here. Instead, reply with a plain text message. Tailor the question based on the issue type they picked in Step 1:

- If **Task**: "Describe what needs to be done."
- If **Bug**: "Describe the bug — steps to reproduce, expected behavior, and actual behavior."
- If **Epic**: "Describe the goal, scope, and success criteria for this epic."
- If **Story**: "Describe the story as: As a [who], I want [what], so that [why]."

Wait for the user to type their answer as a normal chat message.

### Step 4 — Ask for acceptance criteria

Do NOT use AskUserQuestion here. Instead, reply with a plain text message asking:

> "What are the acceptance criteria for this ticket? (List the conditions that must be met for this to be considered done.)"

Wait for the user to type their answer as a normal chat message. When saving, convert the user's input into clean bullet points.

### Step 5 — Ask for priority

Use `AskUserQuestion` with these 4 options:

- **Highest** — System down, data loss, no workaround
- **High** — Major feature broken, workaround exists but painful
- **Medium** — Important but not urgent
- **Low** — Nice to have, minor improvement

### Step 6 — Review and confirm

Rephrase all user inputs for clarity and professionalism before displaying. Convert acceptance criteria into clean bullet points. Display the collected info back to the user in a clean format:

```
Space:       Aqua Claude Docs (MD)
Type:        [selected type]
Summary:     [rephrased summary]
Priority:    [selected priority]

Description:
[rephrased description]

Acceptance Criteria:
- [criterion 1]
- [criterion 2]
- ...
```

Then use `AskUserQuestion` to ask for confirmation with these options:

- **Yes, create it** — Looks good, create the ticket in JIRA
- **No, let me edit** — I want to change something

If the user selects **"No, let me edit"**, ask which field they want to change (type, summary, description, acceptance criteria, or priority), collect the new value, then show the review again. Repeat until they confirm.

### Step 7 — Create the JIRA ticket

Only after the user confirms, create the ticket directly in JIRA using the `createJiraIssue` MCP tool with the following details:

- **Project key:** `MD` (Aqua Claude Docs)
- **Issue type:** as selected by the user
- **Summary:** rephrased summary
- **Description:** rephrased description, followed by the acceptance criteria formatted as bullet points under an "Acceptance Criteria" heading
- **Priority:** as selected by the user

After successful creation, display the ticket key and link to the user.
