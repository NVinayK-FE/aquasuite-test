# Implementation Protocol

<!-- ============================================================
     SHARED PROTOCOL: Ticket/Task Implementation
     ============================================================
     Used by: jira-executor, jira-batch-executor,
              jira-epic-orchestrator (execute modes)
     
     Purpose: Standard flow for implementing any ticket or task.
     Ensures consistency across all execution skills.
     
     Flow: Understand → Spec-First Check → Plan → Confirm → Implement → Verify
     ============================================================ -->

## Section IDs

**Prefix:** `impl`

| ID | Section |
|---|---|
| `impl.1.present-understanding` | Present Understanding |
| `impl.2.transition-progress` | Transition to In Progress |
| `impl.3.spec-first-check` | Run Spec-First Protocol |
| `impl.4.implementation-plan` | Present Implementation Plan |
| `impl.5.implement` | Implement |
| `impl.6.jira-update` | JIRA Update |

## Standard Implementation Flow

### 1. Present Understanding {#impl.1.present-understanding}

Fetch and present your understanding of the ticket/task:

```
Ticket:      [KEY] — [Summary]
Type:        [Issue Type]
Priority:    [Priority]
Status:      [Current Status]

My Understanding:
[Rephrase requirements — what, why, expected outcome]

Acceptance Criteria:
- [criterion 1]
- [criterion 2]
```

Confirm using Human Confirmation Protocol (see `human-confirmation-protocol.md`):

- **Yes, that's correct** — Proceed
- **Partially correct, let me clarify** — Some parts need correction
- **I want to add more context** — Additional details to include

### 2. Transition to In Progress {#impl.2.transition-progress}

Transition the ticket to **In Progress** using `transitionJiraIssue`. Inform the user.

### 3. Run Spec-First Protocol {#impl.3.spec-first-check}

**Read and follow `.claude/skills/jira/_shared/references/spec-first-protocol.md`.**

This is a **hard gate** — no code is written until all required specs are confirmed. The protocol performs a deep audit at file AND section level:

1. **Identify Required Specs** — Analyze ALL spec categories (naming, architecture, config, API, DB, business rules) for what this task needs
2. **Deep Scan** — Read each spec file, check section IDs (`<!-- id: X.Y -->`), assess completeness
3. **Gap Report** — Present all missing files, missing sections, and incomplete sections; ask for human input and additions
4. **Create/Update Specs** — Interactively create missing specs and update incomplete ones with human confirmation at each step
5. **Final Confirmation** — Present complete spec coverage table; only proceed after human confirms all specs are ready

### 4. Present Implementation Plan {#impl.4.implementation-plan}

Based on ticket requirements and confirmed specs, present a concrete plan:

```
Implementation Plan:

1. [File path — new/modified]
   Action: [create / modify / delete]
   Impact: [what this change does]
   
   Code snippet:
   [key code changes]

2. [File path]
   ...
```

Show **only** what the ticket and specs require. If you believe something additional is needed, list it separately as a suggestion.

Confirm:

- **Yes, proceed with this** — Looks good, go ahead
- **No, let me adjust** — I want to change the approach
- **Looks good but I want to add more details** — Additional constraints before proceeding

### 5. Implement {#impl.5.implement}

Execute code changes exactly as confirmed. After completion:

```
Implementation Complete:

Files changed:
- [file path] — [what was done]

Acceptance Criteria Check:
✅ [criterion 1] — [how met]
✅ [criterion 2] — [how met]
```

Confirm:

- **Yes, looks good** — Satisfied with changes
- **No, something needs fixing** — Issue needs correction
- **Run it again with modifications** — Adjust and re-implement

### 6. JIRA Update {#impl.6.jira-update}

Add a comment to the ticket with:
- Implementation summary
- Files changed
- Acceptance criteria status
- Any observations, risks, or follow-up items

Ask about status transition:

- **Mark as Done** — Transition to Done
- **Keep In Progress** — Leave in current status (e.g., awaiting review)
- **Custom status** — Specify status
