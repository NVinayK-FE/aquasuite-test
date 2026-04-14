---
name: jira-flow-creator
description: >
  MANDATORY TRIGGERS: "start flow-creator", "create flow", "document flow",
  "flow documentation", "create e2e", "create test cases", "forgot password flow",
  "login flow", "checkout flow", or any request to document a user flow with
  test scenarios. Use this skill to create or update flow documentation (flow.md),
  E2E test specs (e2e.yaml), and unit test specs (unit.yaml) for any user-facing
  or system flow. Works standalone or integrated into JIRA ticket execution.
---

<!-- ============================================================
     FLOW CREATOR
     ============================================================
     Standalone skill for creating/updating flow documentation.
     
     Trigger: `start flow-creator`
     
     Also used as an integrated protocol during:
       - jira-executor (Step 3b)
       - jira-batch-executor (per ticket)
       - jira-epic-orchestrator (execute modes)
     
     Outputs (per flow):
       .flows/<module>/<flow-name>/
         flow.md       — Flow definition with all scenarios
         e2e.yaml      — E2E test scenarios
         unit.yaml     — Unit test case specs
     
     The .flows/ folder is organized by module, mirroring the
     code's module structure for easy navigation.
     
     Shared protocols used:
       - flow-protocol.md              → Core flow documentation protocol
       - human-confirmation-protocol.md → Enhanced confirmations
     ============================================================ -->

# Flow Creator

Create or update flow documentation interactively. This skill generates three files for each user flow: a flow definition (MD), E2E test scenarios (YAML), and unit test case specs (YAML).

**Project:** ACD (Aqua Claude Dev)

**Shared Protocols:** This skill references reusable protocols in `.claude/skills/jira/_shared/references/`. Read the relevant protocol file when a step says "follow [protocol-name]".

- `flow-protocol.md` — Core flow documentation protocol (all steps)
- `human-confirmation-protocol.md` — Enhanced confirmation patterns (all steps)

## Section IDs

**Prefix:** `fc`

When starting each step, display the section ID:

```
▶ [fc.N.name] — Step title
```

| ID | Step |
|---|---|
| `fc.1.determine-mode` | Determine Mode (New or Update) |
| `fc.2.flow-identity` | Flow Identity |
| `fc.3.flow-definition` | Flow Definition (Interactive) |
| `fc.4.generate-flow-md` | Generate flow.md |
| `fc.5.generate-e2e-yaml` | Generate e2e.yaml |
| `fc.6.generate-unit-yaml` | Generate unit.yaml |
| `fc.7.save-register` | Save and Register |
| `fc.8.git-commit` | Git Commit |

## Output Structure

```
.flows/
  <module>/
    <flow-name>/
      flow.md       — Step-by-step flow with conditions and all scenarios
      e2e.yaml      — Structured E2E test scenarios
      unit.yaml     — Unit test case specs by function/method
```

The `.flows/` folder mirrors the code's module structure. For example: `.flows/auth/forgot-password/`, `.flows/billing/payment-checkout/`.

---

## Flow

### Step 1 — Determine Mode (New or Update) {#fc.1.determine-mode}

Check if the user specified a flow to update or wants to create a new one.

If updating, scan `.flows/` for existing flows and present:

```
Existing Flows:

  auth/
    ├── forgot-password/  (flow.md, e2e.yaml, unit.yaml)
    └── login/            (flow.md, e2e.yaml)

  billing/
    └── payment-checkout/ (flow.md, e2e.yaml, unit.yaml)
```

Use `AskUserQuestion`:

- **Create a new flow** — Start from scratch
- **Update an existing flow** — Modify an existing flow's documentation or tests
- **I want to add more context** — Additional details before deciding

If updating, ask which flow and which files to update (flow.md, e2e.yaml, unit.yaml, or all).

### Step 2 — Flow Identity {#fc.2.flow-identity}

**Follow Step 2 of the Flow Protocol** in `.claude/skills/jira/_shared/references/flow-protocol.md`.

Ask one at a time (plain text, not AskUserQuestion — these are open-ended):

1. "Which module does this flow belong to?" (e.g., auth, billing, user-management, notification)
2. "What's the flow name?" (e.g., forgot-password, user-registration, payment-checkout)
3. "Describe the flow at a high level — what does the user accomplish?"

If a JIRA ticket is associated, mention it: "I see this relates to [TICKET-KEY]. I'll include the ticket reference in the flow documentation."

Use `AskUserQuestion` to confirm:

- **Yes, that's right** — Proceed with flow definition
- **Let me adjust** — I want to change the module or name
- **I want to add more context** — Additional background first

### Step 3 — Flow Definition (Interactive) {#fc.3.flow-definition}

**Follow Step 3 of the Flow Protocol** in `.claude/skills/jira/_shared/references/flow-protocol.md`.

Walk through the flow step by step with the user:

1. "What is the entry point / trigger for this flow?" (e.g., user clicks 'Forgot Password' link)
2. For each step: "What happens next?" and "Are there any conditions or branches here?"
3. Continue until the user says the flow is complete

For each branch/condition, explore all paths:

- **Happy path** — The success scenario
- **Error/failure scenarios** — What can go wrong at each step
- **Edge cases** — Timeout, invalid input, concurrent access, rate limiting, etc.
- **Validation rules** — What's validated, how, and what happens on failure

Ask one question at a time. Never batch questions.

### Step 4 — Generate flow.md {#fc.4.generate-flow-md}

**Follow Step 4 of the Flow Protocol** in `.claude/skills/jira/_shared/references/flow-protocol.md`.

Generate the flow document with this structure:

```markdown
# [Flow Name]

**Module:** [module]
**JIRA Ticket:** [KEY] (if applicable)
**Last Updated:** [date]

## Overview
[High-level description of what this flow achieves]

## Actors
- [Actor 1] — [role in the flow]
- [Actor 2] — [role in the flow]

## Preconditions
- [What must be true before this flow starts]

## Flow Steps

### Step 1 — [Step name]
**Action:** [What happens]
**System:** [How the system responds]
**Validation:** [Rules applied]

  → If [condition]: go to Step 2a
  → If [other condition]: go to Step 2b

### Step 2a — [Happy path step]
...

### Step 2b — [Error path step]
...

## Scenarios

### Happy Path
1. [Step summary] → [outcome]
2. [Step summary] → [outcome]
...

### Error: [Error name]
1. [Step summary] → [error outcome]
...

### Edge Case: [Edge case name]
...

## Business Rules
- [Rule 1]
- [Rule 2]

## Postconditions
- [System state after successful completion]
- [System state after failure]
```

Present the generated flow.md and confirm:

- **Yes, flow document is correct** — Save and continue
- **No, let me adjust** — Modify the flow
- **I want to add more details** — Additional scenarios or rules

### Step 5 — Generate e2e.yaml {#fc.5.generate-e2e-yaml}

**Follow Step 5 of the Flow Protocol** in `.claude/skills/jira/_shared/references/flow-protocol.md`.

Generate E2E test scenarios based on the flow document:

```yaml
flow: [flow-name]
module: [module]
ticket: [JIRA-KEY]  # if applicable
generated: [date]

scenarios:
  - name: "Happy path — [description]"
    tags: [happy-path]
    preconditions:
      - "[condition 1]"
      - "[condition 2]"
    steps:
      - action: "[user action]"
        expected: "[system response]"
      - action: "[next action]"
        expected: "[expected result]"
    postconditions:
      - "[final state assertion]"

  - name: "Error — [error description]"
    tags: [error, negative]
    preconditions:
      - "[setup for error scenario]"
    steps:
      - action: "[action that triggers error]"
        expected: "[error response]"
    postconditions:
      - "[state after error]"

  - name: "Edge case — [description]"
    tags: [edge-case]
    # ...
```

If **updating** an existing e2e.yaml, load the existing file, show what's new/changed, and present a diff before writing.

Present and confirm:

- **Yes, E2E scenarios are correct** — Save and continue
- **No, let me adjust** — Modify scenarios
- **I want to add more test cases** — Additional scenarios to include

### Step 6 — Generate unit.yaml {#fc.6.generate-unit-yaml}

**Follow Step 6 of the Flow Protocol** in `.claude/skills/jira/_shared/references/flow-protocol.md`.

Generate unit test specs organized by function/method:

```yaml
flow: [flow-name]
module: [module]
ticket: [JIRA-KEY]  # if applicable
generated: [date]

unit_tests:
  - function: "[functionName]"
    description: "[What this function does in the flow]"
    cases:
      - name: "[test case name]"
        input:
          param1: "[value]"
          param2: "[value]"
        expected:
          result: "[expected return]"
        
      - name: "[error case name]"
        input:
          param1: "[invalid value]"
        expected:
          error: "[ERROR_CODE]"
          message: "[error message]"
    mocks:
      - "[dependency to mock]"

  - function: "[nextFunction]"
    # ...
```

If **updating** an existing unit.yaml, load the existing file, show what's new/changed, and present a diff before writing.

Present and confirm:

- **Yes, unit test specs are correct** — Save and continue
- **No, let me adjust** — Modify test cases
- **I want to add more test cases** — Additional functions or cases to cover

### Step 7 — Save and Register {#fc.7.save-register}

Save all files to `.flows/<module>/<flow-name>/`:

```
Files Created/Updated:

  .flows/[module]/[flow-name]/
    ├── flow.md     — [created/updated] ([N] steps, [M] scenarios)
    ├── e2e.yaml    — [created/updated] ([N] test scenarios)
    └── unit.yaml   — [created/updated] ([N] functions, [M] test cases)
```

Register the flow in root `CLAUDE.md` under **Registered Flows** (create the section if it doesn't exist):

| ID | File | Description |
|---|---|---|
| `[module].[flow-name]` | `.flows/[module]/[flow-name]/flow.md` | [one-line description] |

Use `AskUserQuestion`:

- **Yes, all saved correctly** — Proceed to git
- **Wait, let me review** — I want to check the files first
- **I want to adjust something** — Go back and modify

### Step 8 — Git Commit {#fc.8.git-commit}

Flow files should be committed separately from any implementation code.

Use `AskUserQuestion`:

- **Yes, commit and push** — Commit flow files and push
- **Commit only, don't push** — Save locally
- **No, skip git** — Leave uncommitted

If committing, use message format: `[KEY]: Add/Update [flow-name] flow documentation` (or without ticket key if standalone).

---

## Important Principles

- **One question at a time.** Flow definition is inherently interactive. Never batch questions.
- **Cover all paths.** Happy path is never enough — always explore errors, edge cases, and validation failures.
- **Module-aligned structure.** Flows mirror the codebase's module organization for discoverability.
- **Update-friendly.** When updating existing flows, always show what changed before saving.
- **Test coverage matters.** E2E scenarios should cover every path in the flow. Unit tests should cover every function involved.
- **Never originate substance.** All flow logic, conditions, and business rules come from the user. You structure and format — you don't decide.
- **Register everything.** Every flow gets registered in CLAUDE.md for discoverability.
