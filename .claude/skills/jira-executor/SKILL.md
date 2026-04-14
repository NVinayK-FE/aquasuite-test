---
name: jira-executor
description: >
  MANDATORY TRIGGERS: "start jira-executor", "execute ticket", "work on ticket",
  "implement ticket", "pick up ticket", or any JIRA ticket key like ACD-4 or MD-12.
  Starts a disciplined, step-by-step JIRA ticket implementation workflow.
  Do NOT scan or read any other project files until the flow explicitly requires it
  in Step 3.
---

# JIRA Executor

Guide the user through implementing a JIRA ticket step by step. Every step requires explicit human confirmation before proceeding. Never skip steps or combine them.

## Flow

### Step 1 — Fetch and present ticket understanding

Fetch the JIRA ticket using the `getJiraIssue` MCP tool. Read the ticket summary, description, and acceptance criteria thoroughly.

Present your full understanding of the ticket back to the user in a clear format:

```
Ticket:      [KEY] — [Summary]
Type:        [Issue Type]
Priority:    [Priority]
Status:      [Current Status]

My Understanding:
[Rephrase the ticket requirements in your own words — what needs to be done, why, and what the expected outcome is]

Acceptance Criteria:
- [criterion 1]
- [criterion 2]
- ...
```

Then use `AskUserQuestion` to confirm:

- **Yes, that's correct** — Your understanding is accurate
- **No, let me clarify** — I need to add more context or correct something

If the user selects **"No, let me clarify"**, collect their clarification, update your understanding, and present it again. Repeat until confirmed.

### Step 2 — Transition ticket to In Progress

Once the user confirms your understanding, transition the JIRA ticket status to **In Progress** using the `transitionJiraIssue` MCP tool. Inform the user that the ticket is now in progress.

### Step 3 — Identify relevant spec files and sections

Scan all `.spec` markdown files in the project. For each file that is relevant to implementing this ticket, list it with its sections and IDs:

```
Relevant Spec Files:

1. [file path]
   - [section-id] — [section title/description]
   - [section-id] — [section title/description]

2. [file path]
   - [section-id] — [section title/description]
```

If no spec files are relevant, state that clearly.

Then use `AskUserQuestion` to confirm:

- **Yes, this is correct** — These are the right spec files and sections
- **No, let me adjust** — I want to add or remove sections
- **Guide me** — Show me all available spec files so I can pick

If the user selects **"No, let me adjust"**, ask which sections to add or remove, then present the updated list. Repeat until confirmed.

If the user selects **"Guide me"**, list ALL available `.spec` markdown files with ALL their sections and IDs. Let the user pick which ones are relevant. Then present the final selection for confirmation.

### Step 4 — Present implementation plan with code snippets

Based on the ticket requirements and the confirmed spec sections, present a concrete implementation plan showing **only** what the ticket and specs require — nothing more. Do not add convenience files (e.g., `.gitkeep`, `.gitignore`, `README.md`), boilerplate, or supplementary tooling unless explicitly stated in the ticket's acceptance criteria or the confirmed spec sections. If you believe something additional is needed, list it separately as a suggestion and let the human decide.

Present the plan in this format:

```
Implementation Plan:

1. [File path — new/modified]
   Action: [create / modify / delete]
   Impact: [what this change does]
   
   Code snippet:
   [show the key code changes — not the full file, just the relevant parts]

2. [File path — new/modified]
   Action: [create / modify / delete]
   Impact: [what this change does]
   
   Code snippet:
   [show the key code changes]

...
```

Then use `AskUserQuestion` to confirm:

- **Yes, proceed** — Looks good, start implementing
- **No, let me adjust** — I want to change the approach

If the user selects **"No, let me adjust"**, collect their feedback, revise the plan, and present it again. Repeat until confirmed.

### Step 5 — Implement the code

Only after the user confirms the implementation plan, execute the actual code changes. Follow the plan exactly as confirmed. Do not deviate without asking.

After implementation is complete, present a summary of all changes made:

```
Implementation Complete:

Files changed:
- [file path] — [what was done]
- [file path] — [what was done]

Acceptance Criteria Check:
✅ [criterion 1] — [how it was met]
✅ [criterion 2] — [how it was met]
```

### Step 6 — Ask about git push

Use `AskUserQuestion` to ask:

- **Yes, push to repository** — Commit and push the changes
- **No, stop here** — Keep the changes local only

If **Yes**: commit the changes with a meaningful commit message referencing the ticket key (e.g., `ACD-4: Set up modular monolith structure`) and push to the repository.

If **No**: leave the changes as-is without committing or pushing.

### Step 7 — Final feedback and JIRA update

Regardless of the git push decision, provide a final summary of the task:

```
Final Summary:

Ticket:    [KEY] — [Summary]
Status:    [In Progress / Done]
Changes:   [brief list of what was implemented]
Notes:     [any observations, risks, or follow-up items]
```

Then update the JIRA ticket:

1. **Add a comment** to the ticket with the final summary and any major observations using the `addCommentToJiraIssue` MCP tool.
2. If there are any major updates, blockers, or risks discovered during implementation, add those as a separate comment.
3. Do NOT transition the ticket to "Done" — leave it in "In Progress" for the user or reviewer to close after verification.
