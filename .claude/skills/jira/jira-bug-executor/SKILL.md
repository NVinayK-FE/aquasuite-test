---
name: jira-bug-executor
description: >
  MANDATORY TRIGGERS: "start jira-bug-executor", "fix bug", "debug ticket",
  or routed from jira-router for Bug issue types.
  Starts a disciplined bug investigation and fix workflow. Unlike jira-executor
  (which is for tasks/subtasks), this skill prioritises root-cause analysis
  before any code changes. It reproduces the issue, traces the root cause,
  confirms the diagnosis with the user, THEN implements the fix.
  Do NOT scan or read any other project files until the flow explicitly
  requires it in Step 3.
---

# JIRA Bug Executor

Guide the user through investigating and fixing a JIRA bug ticket step by step. Every step requires explicit human confirmation before proceeding. Never skip steps or combine them.

**Key difference from jira-executor:** Bugs require investigation before implementation. Steps 3–5 are a dedicated root-cause analysis phase that must be confirmed by the user before any code is written.

**Shared Protocols:** This skill references reusable protocols in `.claude/skills/jira/_shared/references/`. Read the relevant protocol file when a step says "follow [protocol-name]".

- `human-confirmation-protocol.md` — Enhanced confirmation patterns (all steps)
- `spec-first-protocol.md` — Verify .spec coverage before implementation (Step 6)
- `git-workflow.md` — Git operations (Step 9, Level 1)

## Section IDs

**Prefix:** `bug`

When starting each step, display the section ID:

```
▶ [bug.N.name] — Step title
```

| ID | Step |
|---|---|
| `bug.1.fetch-understand` | Fetch and present bug understanding |
| `bug.2.transition-progress` | Transition ticket to In Progress |
| `bug.3.reproduce` | Reproduce the issue |
| `bug.4.root-cause` | Trace root cause |
| `bug.5.confirm-diagnosis` | Confirm diagnosis with user |
| `bug.6.spec-first-check` | Spec-First Check |
| `bug.7.fix-plan` | Present fix plan |
| `bug.8.implement-fix` | Implement the fix |
| `bug.9.git-workflow` | Git workflow (separate commits) |
| `bug.10.jira-update` | Final feedback and JIRA update |

## Flow

### Step 1 — Fetch and present bug understanding {#bug.1.fetch-understand}

Fetch the JIRA ticket using the `getJiraIssue` MCP tool. Read the ticket summary, description, steps to reproduce, expected behaviour, actual behaviour, and any acceptance criteria thoroughly.

Present your full understanding back to the user:

```
Ticket:      [KEY] — [Summary]
Type:        Bug
Priority:    [Priority]
Status:      [Current Status]

Bug Summary:
[Rephrase what the bug is — what is broken and what impact it has]

Steps to Reproduce:
1. [step 1]
2. [step 2]
...

Expected Behaviour:
[What should happen]

Actual Behaviour:
[What actually happens]

Acceptance Criteria (for the fix):
- [criterion 1]
- [criterion 2]
- ...
```

If the ticket description is missing steps to reproduce, expected/actual behaviour, or other critical details, flag this explicitly and ask the user to fill in the gaps.

Then use `AskUserQuestion` to confirm:

- **Yes, that's correct** — Your understanding is accurate, proceed
- **Partially correct, let me clarify** — Some parts need correction
- **I want to add more context** — There are additional details you should know

If the user selects **"Partially correct"** or **"add more context"**, collect their input, update your understanding, and present it again. Repeat until confirmed.

### Step 2 — Transition ticket to In Progress {#bug.2.transition-progress}

Once the user confirms your understanding, transition the JIRA ticket status to **In Progress** using the `transitionJiraIssue` MCP tool. Inform the user that the ticket is now in progress.

### Step 3 — Reproduce the issue {#bug.3.reproduce}

Before investigating the root cause, attempt to reproduce the issue:

1. **Identify the affected area** — Based on the bug description, locate the relevant files, modules, and code paths.
2. **Trace the execution path** — Follow the code from the entry point (API endpoint, UI component, event handler, etc.) through to where the bug manifests.
3. **Check for tests** — Look for existing tests that should have caught this bug. Note whether tests exist but are passing (meaning they don't cover the scenario) or are missing entirely.

Present your findings:

```
Reproduction Analysis:

Affected Area:
- [file/module] — [why this is involved]
- [file/module] — [why this is involved]

Execution Path:
[entry point] → [step 1] → [step 2] → ... → [where bug manifests]

Existing Test Coverage:
- [test file] — [covers/doesn't cover this scenario]
- No tests found for [specific scenario]
```

Then use `AskUserQuestion` to confirm:

- **Yes, that matches what I see** — Your reproduction analysis is correct
- **Not quite, let me clarify** — The bug manifests differently than described
- **I can provide more clues** — I have additional debugging info

### Step 4 — Trace root cause {#bug.4.root-cause}

Now dig deeper to identify the root cause. This is the critical investigation step:

1. **Read the relevant code carefully** — Don't just skim. Read the functions, their inputs, outputs, and edge cases.
2. **Identify the defect** — What specific line(s), condition(s), or logic error(s) cause the bug?
3. **Understand why it was introduced** — Was it a missing edge case? A wrong assumption? A regression from another change? Check `git log` / `git blame` if helpful.
4. **Assess blast radius** — What else could be affected by this same root cause? Are there similar patterns elsewhere that might have the same bug?

Present your root-cause analysis:

```
Root Cause Analysis:

Defect Location:
- [file:line] — [description of the defect]

Root Cause:
[Clear explanation of WHY the bug happens — not just what is wrong, 
but the underlying reason. E.g., "The function assumes the array is 
never empty, but when no items match the filter, it passes an empty 
array which causes the indexOf call on line 42 to return -1, and 
the subsequent slice(-1) returns the last element instead of nothing."]

How It Was Introduced:
[If determinable — e.g., "This edge case was not handled in the 
original implementation" or "Regression from commit abc123 which 
changed the filter logic"]

Blast Radius:
- [Other places that might be affected]
- [Or: "Isolated to this single code path"]
```

Then use `AskUserQuestion` to confirm:

- **Yes, that's the root cause** — Your diagnosis is correct, proceed to fix
- **Close but not exactly** — The root cause is slightly different
- **No, that's wrong** — Let me point you in the right direction

If the user corrects or redirects, re-investigate and present the updated analysis. **Do NOT proceed to implementation until the user confirms the root cause.**

### Step 5 — Confirm diagnosis with user {#bug.5.confirm-diagnosis}

Present a consolidated diagnosis summary and the proposed fix approach (high level, not code yet):

```
Confirmed Diagnosis:

Bug:         [one-line summary of what's broken]
Root Cause:  [one-line summary of why]
Fix Approach: [high-level description of the fix — e.g., "Add a null check 
              before accessing the property" or "Change the query to use 
              LEFT JOIN instead of INNER JOIN"]
Blast Radius: [isolated / moderate / wide — with brief explanation]
```

Then use `AskUserQuestion`:

- **Yes, proceed with fix** — Diagnosis confirmed, move to implementation
- **I want to adjust the approach** — The diagnosis is right but I want a different fix strategy
- **Go back and investigate more** — I'm not confident in this diagnosis yet

### Step 6 — Spec-First Check {#bug.6.spec-first-check}

**Follow the Spec-First Protocol** in `.claude/skills/jira/_shared/references/spec-first-protocol.md`.

Before writing any code, verify that all `.spec` files and sections required for this fix exist. If the bug reveals a gap in the specs (e.g., an undocumented edge case), this is the time to fill it:

1. Present what's missing to the user
2. Interactively gather the spec details
3. Create the `.spec` file/section
4. Register it in root `CLAUDE.md` under Registered Specs
5. Confirm with the user before proceeding

Only after all required specs are in place, move to the next step.

### Step 7 — Present fix plan {#bug.7.fix-plan}

Based on the confirmed diagnosis and the spec sections, present a concrete fix plan. The plan should include both the fix itself AND any tests to prevent regression:

```
Fix Plan:

1. [File path — modified]
   Action: [modify]
   Fix: [what specifically changes and why]
   
   Code snippet:
   [show the key code changes — before and after]

2. [Test file path — new/modified]
   Action: [create / modify]
   Test: [what scenario this test covers to prevent regression]
   
   Code snippet:
   [show the test code]

...
```

Then use `AskUserQuestion` to confirm:

- **Yes, proceed** — Looks good, start implementing the fix
- **No, let me adjust** — I want to change the approach
- **Looks good but I want to add more details** — Additional constraints before proceeding

If the user adjusts or adds details, revise the plan and present again. Repeat until confirmed.

### Step 8 — Implement the fix {#bug.8.implement-fix}

Only after the user confirms the fix plan, execute the actual code changes. Follow the plan exactly as confirmed. Do not deviate without asking.

After implementation is complete, present a summary:

```
Fix Complete:

Files changed:
- [file path] — [what was fixed]
- [file path] — [test added/updated]

Root Cause Addressed:
✅ [description of the defect] — [how it was fixed]

Regression Tests:
✅ [test name] — [what it verifies]

Acceptance Criteria Check:
✅ [criterion 1] — [how it was met]
✅ [criterion 2] — [how it was met]
```

Then use `AskUserQuestion`:

- **Yes, looks good** — I'm satisfied with the fix
- **No, something needs fixing** — I see an issue that needs correction
- **Run it again with modifications** — I want to adjust and re-implement

### Step 9 — Git workflow {#bug.9.git-workflow}

**Follow the Git Workflow Protocol (Level 1 — Basic)** in `.claude/skills/jira/_shared/references/git-workflow.md`.

<!-- ============================================================
     SEPARATE COMMIT PROTOCOL
     ============================================================
     When a bug fix involves changes to multiple categories of files
     (spec files, fix code, tests), commit each category separately.
     This keeps the git history clean and makes it easy to review
     changes by concern.
     
     Commit order:
       1. Spec files (.spec/)     — if created/updated
       2. Bug fix code            — the actual fix
       3. Tests                   — regression tests
     ============================================================ -->

Before committing, categorize all changed files:

```
Changed Files Summary:

Spec files (.spec/):
- [file] — [created/updated]

Bug fix code:
- [file] — [what was fixed]

Tests:
- [file] — [created/updated]
```

**Commit each category separately, in this order:**

1. **Spec files first** (if any were created/updated):
   - Commit message: `[KEY]: Add/Update spec — [brief description]`
   
2. **Bug fix code second:**
   - Commit message: `[KEY]: Fix — [root cause summary]`

3. **Tests last** (if any were created/updated):
   - Commit message: `[KEY]: Add regression test — [what it covers]`

For each commit, use `AskUserQuestion`:

- **Yes, commit and push** — Commit and push the changes to remote
- **Commit only, don't push** — Save locally but don't push
- **No, skip git** — Leave changes uncommitted

If only one category has changes, just do a single commit.

### Step 10 — Final feedback and JIRA update {#bug.10.jira-update}

Regardless of the git decision, provide a final summary:

```
Final Summary:

Ticket:      [KEY] — [Summary]
Status:      In Progress
Root Cause:  [one-line summary]
Fix:         [one-line summary of what was changed]
Tests:       [added/updated/none]
Specs:       [created/updated/none]
Git:         [N commits — committed/pushed/none]
Blast Radius: [isolated/moderate/wide]
Notes:       [any observations, risks, or follow-up items]
```

Then update the JIRA ticket:

1. **Add a comment** to the ticket with the root-cause analysis and fix summary using the `addCommentToJiraIssue` MCP tool. Include the root cause, what was fixed, and what tests were added.
2. If there are related areas that should be checked or follow-up tickets that should be created, add those as a separate comment.
3. Do **NOT** transition the ticket to "Done" — leave it in "In Progress" for human review. Only the human decides when a ticket is Done.

Use `AskUserQuestion` for final wrap-up:

- **Done, that's all** — End the session
- **I want to add a note to the ticket** — I have additional observations to record
- **Start another ticket** — I want to execute another ticket next
