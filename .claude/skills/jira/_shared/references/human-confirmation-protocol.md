# Human Confirmation Protocol

<!-- ============================================================
     SHARED PROTOCOL: Enhanced Human Confirmation
     ============================================================
     Used by: ALL jira skills
     
     Purpose: Every significant step must give the user rich options
     to confirm, adjust, add details, or pause. The goal is to build
     confidence that things are going smoothly — never rush past
     a step without ensuring the user is comfortable.
     
     This replaces the basic "Yes/No" pattern with a richer set
     of options that always includes "add more details."
     ============================================================ -->

## Section IDs

**Prefix:** `hcp`

| ID | Section |
|---|---|
| `hcp.1.core-principle` | Core Principle |
| `hcp.2.understanding-pattern` | After Presenting Understanding |
| `hcp.3.plan-pattern` | After Presenting a Plan |
| `hcp.4.specs-pattern` | After Generating Specs or Artifacts |
| `hcp.5.implementation-pattern` | After Implementation |
| `hcp.6.destructive-pattern` | Before Any Destructive or Irreversible Action |
| `hcp.7.confidence-checkpoints` | Confidence Checkpoints |

## Core Principle {#hcp.1.core-principle}

Never present just "Yes" and "No" as the only options. Always include a path to add more details or context. The user should feel they are guiding every step, not just rubber-stamping.

## Standard Confirmation Patterns

### After Presenting Understanding (tickets, tasks, requirements) {#hcp.2.understanding-pattern}

Use `AskUserQuestion`:

- **Yes, that's correct** — Your understanding is accurate, proceed
- **Partially correct, let me clarify** — Some parts need correction
- **I want to add more context** — There are additional details you should know

### After Presenting a Plan (implementation plan, task breakdown, story list) {#hcp.3.plan-pattern}

Use `AskUserQuestion`:

- **Yes, proceed with this** — Looks good, go ahead
- **No, let me adjust** — I want to change the approach
- **Looks good but I want to add more details** — I have additional requirements or constraints to include before proceeding

### After Generating Specs or Artifacts (database models, API contracts, etc.) {#hcp.4.specs-pattern}

Use `AskUserQuestion`:

- **Yes, specs are correct** — Save and move on
- **No, let me adjust** — I want to modify something
- **Add more details to these specs** — I have additional fields, constraints, or rules to include

### After Implementation (code written, files changed) {#hcp.5.implementation-pattern}

Use `AskUserQuestion`:

- **Yes, looks good** — I'm satisfied with the changes
- **No, something needs fixing** — I see an issue that needs correction
- **Run it again with modifications** — I want to adjust the approach and re-implement

### Before Any Destructive or Irreversible Action (JIRA creation, git push, status transition) {#hcp.6.destructive-pattern}

Use `AskUserQuestion`:

- **Yes, go ahead** — I confirm this action
- **Wait, let me review first** — I want to check something before committing
- **No, skip this** — Don't perform this action

## Confidence Checkpoints {#hcp.7.confidence-checkpoints}

At key milestones in any flow, insert a confidence checkpoint. This is not a simple "continue?", but a genuine check-in:

```
Quick check before we continue:

We've completed [what was done].
Next up: [what's about to happen].

Everything looking good so far?
```

Use `AskUserQuestion`:

- **Yes, all good — continue** — Everything is on track
- **Hold on, I want to revisit something** — Let me go back and check something
- **Yes, but I want to add something first** — There's additional context before we proceed

### When to Insert Confidence Checkpoints

- After completing a major phase (e.g., all stories defined, all tasks spec'd)
- Before transitioning from planning to execution
- Before the first JIRA ticket creation in a batch
- After the first ticket in a batch (to catch issues early before repeating)
- Before any git push or PR creation
