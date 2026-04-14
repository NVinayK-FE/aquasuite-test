---
name: jira-executor
description: >
  MANDATORY TRIGGERS: "start jira-executor", "execute ticket", "work on ticket",
  "implement ticket", "pick up ticket", or any JIRA ticket key like ACD-4.
  Starts a disciplined, step-by-step JIRA ticket implementation workflow.
  Do NOT scan or read any other project files until the flow explicitly requires it
  in Step 3.
---

# JIRA Executor

Guide the user through implementing a JIRA ticket step by step. Every step requires explicit human confirmation before proceeding. Never skip steps or combine them.

**Shared Protocols:** This skill references reusable protocols in `.claude/skills/jira/_shared/references/`. Read the relevant protocol file when a step says "follow [protocol-name]".

- `human-confirmation-protocol.md` — Enhanced confirmation patterns (all steps)
- `spec-first-protocol.md` — Verify .spec coverage before implementation (Step 3)
- `flow-protocol.md` — Detect and document user flows (Step 3b)
- `git-workflow.md` — Git operations (Step 7, Level 1)

## Section IDs

**Prefix:** `exc`

When starting each step, display the section ID:

```
▶ [exc.N.name] — Step title
```

| ID | Step |
|---|---|
| `exc.1.fetch-understand` | Fetch and present ticket understanding |
| `exc.2.transition-progress` | Transition ticket to In Progress |
| `exc.3.spec-first-check` | Spec-First Check |
| `exc.3b.flow-check` | Flow Check |
| `exc.4.identify-specs` | Identify relevant spec files |
| `exc.5.implementation-plan` | Present implementation plan |
| `exc.6.implement-code` | Implement the code |
| `exc.7.git-workflow` | Git workflow (separate commits) |
| `exc.8.jira-update` | Final feedback and JIRA update |

## Flow

### Step 1 — Fetch and present ticket understanding {#exc.1.fetch-understand}

Fetch the JIRA ticket using the `getJiraIssue` MCP tool. Read the ticket summary, description, and acceptance criteria thoroughly.

Present your full understanding back to the user:

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

- **Yes, that's correct** — Your understanding is accurate, proceed
- **Partially correct, let me clarify** — Some parts need correction
- **I want to add more context** — There are additional details you should know

If the user selects **"Partially correct"** or **"add more context"**, collect their input, update your understanding, and present it again. Repeat until confirmed.

### Step 2 — Transition ticket to In Progress {#exc.2.transition-progress}

Once the user confirms your understanding, transition the JIRA ticket status to **In Progress** using the `transitionJiraIssue` MCP tool. Inform the user that the ticket is now in progress.

### Step 3 — Spec-First Check {#exc.3.spec-first-check}

**Follow the Spec-First Protocol** in `.claude/skills/jira/_shared/references/spec-first-protocol.md`.

Before writing any code, verify that all `.spec` files and sections required for this ticket exist. If any are missing:

1. Present what's missing to the user
2. Interactively gather the spec details
3. Create the `.spec` file/section
4. Register it in root `CLAUDE.md` under Registered Specs
5. Confirm with the user before proceeding

Only after all required specs are in place, move to the next step.

### Step 3b — Flow Check {#exc.3b.flow-check}

**Follow the Flow Protocol** in `.claude/skills/jira/_shared/references/flow-protocol.md`.

After specs are confirmed, analyze the ticket to detect if it involves a user flow (login, registration, forgot-password, checkout, payment, onboarding, etc.). If a flow is detected:

1. Ask the user whether to create/update flow documentation
2. If yes, walk through the full Flow Protocol: gather flow details interactively, generate `flow.md`, `e2e.yaml`, and `unit.yaml`
3. Save files to `.flows/<module>/<flow-name>/`
4. Register the flow in root `CLAUDE.md` under Registered Flows
5. These flow files will be committed separately in Step 7

If no flow is detected, or the user opts to skip, move to the next step.

### Step 4 — Identify relevant spec files {#exc.4.identify-specs}

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

### Step 5 — Present implementation plan {#exc.5.implementation-plan}

Based on the ticket requirements and the confirmed spec sections, present a concrete implementation plan showing **only** what the ticket and specs require — nothing more. Do not add convenience files (e.g., `.gitkeep`, `.gitignore`, `README.md`), boilerplate, or supplementary tooling unless explicitly stated in the ticket's acceptance criteria or the confirmed spec sections. If you believe something additional is needed, list it separately as a suggestion and let the human decide.

Present the plan:

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
- **Looks good but I want to add more details** — Additional constraints before proceeding

If the user adjusts or adds details, revise the plan and present again. Repeat until confirmed.

### Step 6 — Implement the code {#exc.6.implement-code}

Only after the user confirms the implementation plan, execute the actual code changes. Follow the plan exactly as confirmed. Do not deviate without asking.

After implementation is complete, present a summary:

```
Implementation Complete:

Files changed:
- [file path] — [what was done]
- [file path] — [what was done]

Acceptance Criteria Check:
✅ [criterion 1] — [how it was met]
✅ [criterion 2] — [how it was met]
```

Then use `AskUserQuestion`:

- **Yes, looks good** — I'm satisfied with the changes
- **No, something needs fixing** — I see an issue that needs correction
- **Run it again with modifications** — I want to adjust and re-implement

### Step 7 — Git workflow {#exc.7.git-workflow}

**Follow the Git Workflow Protocol (Level 1 — Basic)** in `.claude/skills/jira/_shared/references/git-workflow.md`.

<!-- ============================================================
     SEPARATE COMMIT PROTOCOL
     ============================================================
     When a ticket involves changes to multiple categories of files
     (spec files, flow files, implementation code), commit each
     category separately. This keeps the git history clean and
     makes it easy to review changes by concern.
     
     Commit order:
       1. Spec files (.spec/)     — if created/updated
       2. Flow files (.flows/)    — if created/updated
       3. Implementation code     — the actual code changes
     ============================================================ -->

Before committing, categorize all changed files:

```
Changed Files Summary:

Spec files (.spec/):
- [file] — [created/updated]

Flow files (.flows/):
- [file] — [created/updated]

Implementation code:
- [file] — [created/modified]
```

**Commit each category separately, in this order:**

1. **Spec files first** (if any were created/updated):
   - Commit message: `[KEY]: Add/Update spec — [brief description]`
   
2. **Flow files second** (if any were created/updated):
   - Commit message: `[KEY]: Add/Update [flow-name] flow documentation`

3. **Implementation code last:**
   - Commit message: `[KEY]: [Implementation summary]`

For each commit, use `AskUserQuestion`:

- **Yes, commit and push** — Commit and push the changes to remote
- **Commit only, don't push** — Save locally but don't push
- **No, skip git** — Leave changes uncommitted

If only one category has changes, just do a single commit as before.

### Step 8 — Final feedback and JIRA update {#exc.8.jira-update}

Regardless of the git decision, provide a final summary:

```
Final Summary:

Ticket:    [KEY] — [Summary]
Status:    In Progress
Changes:   [brief list of what was implemented]
Specs:     [created/updated/none]
Flows:     [created/updated/none]
Git:       [N commits — committed/pushed/none]
Notes:     [any observations, risks, or follow-up items]
```

Then update the JIRA ticket:

1. **Add a comment** to the ticket with the final summary and any major observations using the `addCommentToJiraIssue` MCP tool.
2. If there are any major updates, blockers, or risks discovered during implementation, add those as a separate comment.
3. Do **NOT** transition the ticket to "Done" — leave it in "In Progress" for human review. Only the human decides when a ticket is Done.

Use `AskUserQuestion` for final wrap-up:

- **Done, that's all** — End the session
- **I want to add a note to the ticket** — I have additional observations to record
- **Start another ticket** — I want to execute another ticket next
