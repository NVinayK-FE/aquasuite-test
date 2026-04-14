# Flow Protocol

<!-- ============================================================
     SHARED PROTOCOL: Interactive Flow Documentation
     ============================================================
     Used by: jira-executor, jira-batch-executor, jira-epic-orchestrator (execute modes), flow-creator
     
     Purpose: When a ticket involves a user-facing workflow or process,
     create comprehensive flow documentation including:
     - flow.md — Complete flow with all scenarios and decision points
     - e2e.yaml — End-to-end test scenarios
     - unit.yaml — Unit test case specifications
     
     Flow files are stored in .flows/<module>/<flow-name>/ and tracked
     separately from implementation code. Specs are the contract;
     code follows from them.
     ============================================================ -->

## Section IDs

**Prefix:** `flow`

| ID | Section |
|---|---|
| `flow.1.detection` | Flow Detection |
| `flow.2.identity` | Flow Identity |
| `flow.3.definition` | Flow Definition (Interactive) |
| `flow.4.generate-flow-md` | Generate flow.md |
| `flow.5.generate-e2e-yaml` | Generate e2e.yaml |
| `flow.6.generate-unit-yaml` | Generate unit.yaml |
| `flow.7.save-register` | Save and Register |
| `flow.8.commit-flow` | Commit Flow Files Separately |

## When to Use

This protocol is triggered in these contexts:

### During jira-executor or jira-batch-executor
After the Spec-First Check is complete, analyze the ticket requirements. If the ticket involves a **user-facing flow or workflow**, detect it and ask the user whether to create flow documentation:

**Trigger keywords:**
- "flow", "workflow", "process", "journey", "scenario"
- User actions: login, registration, forgot-password, checkout, payment, onboarding, subscription
- Multi-step processes: account creation, order placement, password reset, profile updates

**Decision point:**
Present three options:

- **Yes, create flow documentation** — Proceed with the full flow protocol
- **No, skip flow docs** — Continue with implementation without flow documentation
- **I want to add more context about the flow** — User provides additional details before proceeding

### During epic-orchestrator execution (Mode 3/5)
Same detection per task. If a task in the epic involves a flow, ask the same decision point.

### Standalone via flow-creator skill
User explicitly invokes the flow-creator skill with a module and flow name.

## Protocol Steps

### Step 1 — Flow Detection (skip for standalone) {#flow.1.detection}

Analyze the ticket/task requirements and identify if a user-facing flow is involved. Look for:

- Keywords: "flow", "workflow", "process", "journey", "scenario", "feature"
- User actions: login, register, logout, forgot-password, reset-password, checkout, payment, subscription, onboarding, profile-setup
- Multi-step transactions or wizard-like experiences
- Complex decision trees or error handling paths

**Present the detection:**

```
Flow Detection:

This ticket appears to involve a user flow:
  - Type: [e.g., "Authentication flow"]
  - Key steps: [brief summary]
  - Complexity: [low/medium/high]

Would you like to create flow documentation for this?
```

Use `AskUserQuestion`:

- **Yes, create flow documentation** — Proceed with flow protocol
- **No, skip flow docs** — Continue without flow documentation
- **I want to add more context about the flow** — User provides additional details first

If user provides additional context, record it and then use `AskUserQuestion` again:

- **Now create the documentation** — Proceed with flow protocol
- **Actually, skip it** — Move on without flow documentation

### Step 2 — Flow Identity {#flow.2.identity}

Gather information about the flow. Ask **one at a time**:

**Question 1:** "Which module does this flow belong to? (e.g., auth, billing, user-management, onboarding, checkout)"

Wait for user response. Confirm:

```
Module: [user's response]

Is this correct?
```

Use `AskUserQuestion`:

- **Yes, that's correct** — Proceed to next question
- **No, let me correct it** — User provides different module

**Question 2:** "What's the flow name? (e.g., forgot-password, user-registration, checkout-process)"

Confirm:

```
Flow name: [user's response]

Is this correct?
```

Use `AskUserQuestion`:

- **Yes, that's correct** — Proceed to next question
- **No, let me correct it** — User provides different name

**Question 3:** "Describe the flow at a high level — what does the user accomplish or what problem does this flow solve?"

Confirm:

```
Flow description:
[user's response]

Is this how you'd describe it?
```

Use `AskUserQuestion`:

- **Yes, that's correct** — Proceed to Step 3
- **No, let me refine it** — User provides different description
- **I want to add more details** — User adds additional context before confirming

### Step 3 — Flow Definition (Interactive) {#flow.3.definition}

Walk through the flow step by step, building the complete flow path.

**Question 1:** "What is the entry point / trigger for this flow? (e.g., user clicks 'Forgot Password' button, user navigates to registration page, system detects failed login attempt)"

Record the entry point.

**Continue the flow:**

For each step, ask: "What happens next?" and wait for the user to describe the next step in the flow.

After each step, ask: "Are there any conditions, branches, or error scenarios at this step?"

If yes, explore:

- **Happy path conditions** — What leads to success?
- **Error/failure scenarios** — What goes wrong and how is it handled?
- **Edge cases** — Timeout, invalid input, concurrent access, rate limiting, missing data
- **Validation rules** — What rules are checked at this step?

**Present each branch clearly:**

```
Step [N]: [Step description]

Branches:
  ✓ Success path → [Next step]
  ✗ Error: [error condition] → [Error handling]
  ⚠ Edge case: [edge case] → [Behavior]

Continue to next step?
```

Continue asking "What happens next?" until the user indicates the flow is complete.

Record all steps, decision points, branches, and conditions.

### Step 4 — Generate flow.md {#flow.4.generate-flow-md}

Generate the complete flow document with the following structure:

```markdown
# [Flow Name]

## Overview
- **Module:** [module]
- **Description:** [high-level description]
- **Status:** [Active / In Development]

## Actors
- [Who participates in this flow? e.g., "End User", "System Admin", "API Service"]

## Preconditions
- [What must be true before the flow starts? e.g., "User account exists", "Email verified"]

## Flow Steps

### Happy Path
1. [Step 1 with clear action]
2. [Step 2 with clear action]
3. [Step N with clear action]

### Decision Points
- **[Decision name]** at Step N:
  - Condition A → leads to step X
  - Condition B → leads to step Y

## Scenarios

### Scenario 1: [Happy Path Name]
**Description:** [What succeeds]
**Steps:**
1. [Step with expected result]
2. [Step with expected result]
**Postconditions:**
- [System state after success]

### Scenario 2: [Error/Alternative Name]
**Description:** [What happens]
**Preconditions:**
- [What needs to be true]
**Steps:**
1. [Step]
2. [Step with error handling]
**Postconditions:**
- [System state after error]

## Validation Rules
- [Business rule at step N]
- [Input validation rule]
- [State constraint]

## Postconditions
- [What is true after the flow completes?]
- [What state changes occur?]
```

Present the generated flow.md:

```
Generated flow.md:

[display the markdown]

Ready to save this flow document?
```

Use `AskUserQuestion`:

- **Yes, looks correct** — Save and proceed to Step 5
- **No, let me adjust** — User specifies what to change
- **Add more details to this flow** — User provides additional scenarios or rules

Repeat until user confirms the flow.md is correct.

### Step 5 — Generate e2e.yaml {#flow.5.generate-e2e-yaml}

Generate structured E2E test scenarios based on the flow definition:

```yaml
flow: [flow-name]
module: [module]
description: "[Flow description]"

scenarios:
  - name: "[Scenario Name]"
    description: "[What this scenario tests]"
    preconditions:
      - "[Precondition 1]"
      - "[Precondition 2]"
    steps:
      - action: "[User action or system action]"
        expected: "[Expected outcome or state change]"
      - action: "[Next action]"
        expected: "[Expected outcome]"
    postconditions:
      - "[System state after scenario]"
    
  - name: "[Alternative Scenario]"
    # ... more scenarios for error paths, edge cases, etc.
```

Present the generated e2e.yaml:

```
Generated e2e.yaml:

[display the YAML]

Ready to save these E2E test scenarios?
```

Use `AskUserQuestion`:

- **Yes, looks correct** — Save and proceed to Step 6
- **No, let me adjust** — User specifies what to change
- **Add more test scenarios** — User provides additional scenarios

Repeat until user confirms e2e.yaml is correct.

### Step 6 — Generate unit.yaml {#flow.6.generate-unit-yaml}

Generate unit test case specifications organized by function or method:

```yaml
flow: [flow-name]
module: [module]
description: "[Flow description]"

unit_tests:
  - function: "[Function or method name]"
    description: "[What this function does]"
    cases:
      - name: "[Test case name]"
        description: "[What this case tests]"
        input:
          [input_param]: "[value or structure]"
        expected:
          [output_param]: "[expected value]"
        mocks:
          - "[Dependency to mock]"
      
      - name: "[Another test case]"
        # ... more cases
    
  - function: "[Next function]"
    # ... more functions and their test cases
```

Present the generated unit.yaml:

```
Generated unit.yaml:

[display the YAML]

Ready to save these unit test specifications?
```

Use `AskUserQuestion`:

- **Yes, looks correct** — Save and proceed to Step 7
- **No, let me adjust** — User specifies what to change
- **Add more test cases** — User provides additional unit test cases

Repeat until user confirms unit.yaml is correct.

### Step 7 — Save and Register {#flow.7.save-register}

After all three files (flow.md, e2e.yaml, unit.yaml) are confirmed:

**Step 7a — Create directory structure:**

```
.flows/
  [module]/
    [flow-name]/
      flow.md
      e2e.yaml
      unit.yaml
```

**Step 7b — Save files:**

Create or update the three files. If updating existing files, show a summary of changes:

```
Files to save:

[NEW] .flows/auth/forgot-password/flow.md (2.3 KB)
[NEW] .flows/auth/forgot-password/e2e.yaml (1.8 KB)
[NEW] .flows/auth/forgot-password/unit.yaml (2.1 KB)

Ready to save?
```

Use `AskUserQuestion`:

- **Yes, save these files** — Proceed with saving
- **Wait, let me review first** — User wants to check something
- **No, don't save** — Skip saving

**Step 7c — Register in CLAUDE.md:**

Check if a "Registered Flows" section exists in the root `CLAUDE.md`. If not, create it.

Add an entry following this format:

```markdown
| ID | Module | Flow Name | Files | Description |
|---|---|---|---|---|
| `flow.<module>.<flow-name>` | `<module>` | `<flow-name>` | [`.flows/<module>/<flow-name>/`](.flows/<module>/<flow-name>/) | `<description>` |
```

Present the registration:

```
Flow Registered:

ID: flow.auth.forgot-password
Module: auth
Flow name: forgot-password
Files: .flows/auth/forgot-password/
Description: [description]

Registered in CLAUDE.md: ✅
```

### Step 8 — Commit Flow Files Separately {#flow.8.commit-flow}

Flow documentation should be committed separately from implementation code.

Present the commit option:

```
Flow files are ready to commit separately.

Commit message format:
  [TICKET-KEY]: Add flow documentation for <module>/<flow-name>

Example:
  ACD-42: Add flow documentation for auth/forgot-password
```

Use `AskUserQuestion`:

- **Yes, commit these files** — Create a commit with the flow files
- **Skip the commit** — Don't commit yet, continue with implementation
- **Commit with custom message** — User provides a different commit message

If committing, use git workflow to:

1. Stage the flow files: `git add .flows/[module]/[flow-name]/ CLAUDE.md`
2. Create the commit with the provided message
3. Present confirmation

If user skips the commit, note that flow files are ready to be committed and can be committed later as part of the implementation.

## Output Structure

All flow files are organized as follows:

```
.flows/
  <module>/
    <flow-name>/
      flow.md       — Complete flow documentation
      e2e.yaml      — End-to-end test scenarios
      unit.yaml     — Unit test case specifications
```

**Directory naming:**
- `<module>`: kebab-case, matches the system module (e.g., `auth`, `billing`, `checkout`)
- `<flow-name>`: kebab-case, descriptive name (e.g., `forgot-password`, `user-registration`, `payment-processing`)

## Integration Points

### When running jira-executor or jira-batch-executor:

After Spec-First Check → Run Flow Detection → If flow detected, proceed with flow protocol → Then continue with implementation.

### When running epic-orchestrator (Mode 3/5):

After Spec-First Check for each task → Run Flow Detection → If flow detected, proceed with flow protocol → Then continue with task implementation.

### Git workflow after flow protocol:

Flow files should be committed before or after implementation code, depending on user preference. If committing separately, note that the implementation commit will come after.
