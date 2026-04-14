<!-- ============================================================
     MODE 3 — EXECUTE EPIC
     ============================================================
     Trigger: execute epic <EPIC-KEY>
     Example: execute epic ACD-5
     
     Purpose: Execute an existing epic by implementing all pending
     stories and tasks one by one. Each story follows a complete
     workflow: Spec-First check, Flow detection, task-by-task
     implementation, separate commits, and JIRA updates.
     
     This mode implements ONLY pending/unfinished work. Already
     Done tickets are skipped.
     
     5 Stages:
       1. Load Epic Context — Fetch epic, stories, tasks, spec file
       2. Story-by-Story Execution — Spec-first → Flow → Tasks
       3. Git Commit per Story — Separate commits (spec, flow, code)
       4. Next Story — Ask to continue/stop
       5. Epic Completion — When all done, close epic and summarize
     
     Shared Protocols Used:
       - spec-first-protocol.md — Before each story
       - flow-protocol.md — Flow detection per task
       - implementation-protocol.md — Per-task execution
       - human-confirmation-protocol.md — All user confirmations
       - git-workflow.md Level 2 — Commit per story
     ============================================================ -->

# Mode 3 — Execute Epic

**Trigger:** `execute epic <EPIC-KEY>` (e.g., `execute epic ACD-5`)

Execute an existing JIRA epic by implementing all pending stories and tasks. This mode runs a complete spec-first, flow-aware, commit-per-story workflow.

**Project:** ACD (Aqua Claude Dev)

**Key Principles:**

- Only implement pending/unfinished tickets — skip Done items
- Spec-first before any implementation
- Detect and document user flows per task
- Commit separately: spec → flow → code
- Update JIRA statuses as work progresses
- One story at a time, with full user confirmation

## Section IDs

**Prefix:** `exe`

When starting each stage, display the section ID:

```
▶ [exe.N.name] — Stage title
```

| ID | Stage |
|---|---|
| `exe.1.load-epic-context` | Load Epic Context |
| `exe.2.story-execution` | Story-by-Story Execution |
| `exe.3.git-commit` | Git Commit (Per Story) |
| `exe.4.next-story` | Next Story |
| `exe.5.epic-completion` | Epic Completion |

---

## Stage 1 — Load Epic Context {#exe.1.load-epic-context}

### Step 1.1 — Fetch Epic and Child Issues

Use `getJiraIssue` to fetch the epic by the provided key. Then fetch all child stories and tasks using `searchJiraIssuesUsingJql` or by reading issue links.

Extract:
- Epic key, title, description
- All child stories (status, summary, acceptance criteria)
- All child tasks (status, summary, parent story)
- Epic current status and assignee

### Step 1.2 — Load Spec File

Look for the epic's spec file:

- If epic type is **Epic** → search `.spec/epics/<name>.md`
- If epic type is **Initiative** → search `.spec/initiatives/<name>.md`

If found, read the file. If not found, note it and proceed.

### Step 1.3 — Filter to Pending Only

From all child stories and tasks, identify which are pending (status != Done).

Present the overview:

```
Epic:        [KEY] — [Title]
Current Status: [Status]

Pending work:
  Stories:   [count] unfinished
  Tasks:     [count] unfinished total
  
Spec file:   [Found / Not Found]
  Location:  [.spec/epics/... or .spec/initiatives/...]
  
Ready to start implementation?
```

Use `AskUserQuestion`:

- **Yes, let's start** — Begin with the first pending story
- **No, I want to review the plan first** — Show the full pending list with details
- **Add more context first** — Provide additional notes about the epic before starting

If user wants to review, list all pending stories with their tasks and brief summaries.

Then repeat the question above.

---

## Stage 2 — Story-by-Story Execution {#exe.2.story-execution}

For each pending story (in order):

### Step 2.1 — Transition Story to In Progress

Use `transitionJiraIssue` to move the story from its current status to **In Progress**. Inform the user:

```
[STORY-KEY]: Now In Progress
```

### Step 2.2 — Run Spec-First Protocol

**Read and follow `.claude/skills/jira/_shared/references/spec-first-protocol.md`.**

Apply the Spec-First Protocol for this story:

1. Identify required specs (database, API, architecture, naming conventions, business rules)
2. Scan existing `.spec/` files
3. Present findings (what's covered, what's missing)
4. If missing specs, ask user whether to create them or skip
5. Create missing specs if user agrees
6. Confirm all specs are ready

After spec coverage is confirmed, proceed to next step.

### Step 2.3 — Run Flow Detection (Flow-Aware Task Check)

Analyze the story's pending tasks to detect if any involve a **user-facing flow or workflow**.

Use the first section of `.claude/skills/jira/_shared/references/flow-protocol.md` (Step 1 — Flow Detection):

Look for keywords in the story and task descriptions:
- "flow", "workflow", "process", "journey", "scenario"
- User actions: login, register, logout, forgot-password, checkout, payment, subscription, onboarding, etc.
- Multi-step transactions or wizard-like experiences

**Present detection:**

```
Flow Detection:

Analyzing story [KEY] and its tasks for user flows...

[Summary of findings: "This story involves a user registration flow with password reset recovery" or "No user flows detected in this story"]
```

Use `AskUserQuestion`:

- **Yes, create flow documentation** — Proceed with flow protocol for this story
- **No, skip flow docs** — Continue without flow documentation
- **I want to add more context about this flow** — User provides additional details before proceeding

If user wants to add context, gather it, then ask again:

- **Now create the documentation** — Proceed with flow protocol
- **Actually, skip it** — Move to task implementation without flow docs

**If creating flow documentation:**

Follow `.claude/skills/jira/_shared/references/flow-protocol.md` fully (Steps 2-7). This generates:

- `.flows/<module>/<flow-name>/flow.md` — Complete flow with all scenarios
- `.flows/<module>/<flow-name>/e2e.yaml` — E2E test scenarios
- `.flows/<module>/<flow-name>/unit.yaml` — Unit test specs

After flow files are generated and confirmed, return to this step.

### Step 2.4 — Execute Each Pending Task

For each pending task in the story (in order):

#### 2.4a — Present Task Understanding

Fetch the task and present your understanding:

```
Task:     [TASK-KEY] — [Summary]
Type:     [Issue Type]
Priority: [Priority]
Status:   [Current Status]

My Understanding:
[Rephrase what the task requires — what, why, expected outcome]

From Story Plan:
[Technical specs from the epic's spec file or description]

Implementation Approach:
[How you plan to implement this based on specs and story context]

Acceptance Criteria:
- [criterion 1]
- [criterion 2]
```

Confirm using **Human Confirmation Protocol** (see `human-confirmation-protocol.md`):

Use `AskUserQuestion`:

- **Yes, let's implement this** — Proceed with implementation
- **Partially correct, let me clarify** — Some parts need correction
- **I want to add more context** — Additional details to include before implementing
- **Skip this task for now** — Move to next task without implementing

If user provides clarification or context, update your understanding and ask again.

If user wants to skip, note it and move to next task in the story.

#### 2.4b — Implement the Task

Transition the task to **In Progress** using `transitionJiraIssue`.

**Read and follow `.claude/skills/jira/_shared/references/implementation-protocol.md` steps 3-5:**

- Step 3 — Spec-First Check (verify all required specs are available)
- Step 4 — Present Implementation Plan (file changes, code snippets)
- Step 5 — Implement (execute code changes)

Present the implementation results:

```
Implementation Complete:

Files changed:
- [file path] — [what was done]
- [file path] — [what was done]

Acceptance Criteria Check:
✅ [criterion 1] — [how met]
✅ [criterion 2] — [how met]
```

Confirm:

Use `AskUserQuestion`:

- **Yes, looks good** — Satisfied with changes
- **No, something needs fixing** — Issue needs correction
- **Run it again with modifications** — Adjust and re-implement

If user needs changes, make adjustments and re-present.

#### 2.4c — Update JIRA and Transition Task to Done

Add a comment to the task with:
- Implementation summary
- Files changed
- Acceptance criteria status

Use `transitionJiraIssue` to move task to **Done**.

Present confirmation:

```
[TASK-KEY]: Marked Done
Comment added with implementation details.
```

#### 2.4d — Repeat for Each Pending Task

Return to step 2.4a for the next pending task in the story.

If all tasks in the story are done, proceed to 2.5.

### Step 2.5 — Transition Story to Done

After all tasks are implemented, use `transitionJiraIssue` to move the story to **Done**.

Add a comment to the story:

```
All tasks completed.
- [TASK-KEY] ✓
- [TASK-KEY] ✓
- [TASK-KEY] ✓

Story ready for review and integration.
```

---

## Stage 3 — Git Commit (Per Story) {#exe.3.git-commit}

After a story is complete (all tasks done, story transitioned to Done), perform a categorized commit following the **separate commit protocol**.

### Step 3.1 — Categorize Changed Files

Review all files changed during this story's implementation and categorize them:

- **Spec files** — Files under `.spec/` or `.flows/`
  - Spec files: `.spec/schemas/`, `.spec/contracts/`, `.spec/rules/`, etc.
  - Flow files: `.flows/<module>/<flow-name>/` (flow.md, e2e.yaml, unit.yaml)
- **Implementation code** — Everything else

### Step 3.2 — Commit Separately (If Multiple Categories)

**Read and follow `.claude/skills/jira/_shared/references/git-workflow.md` Level 2.**

If changes span multiple categories, commit separately in this order:

#### Commit 1 — Specs and Flows (if any)

Combine all spec and flow file changes into ONE commit:

```
Message: ACD-[STORY-KEY]: Add/Update spec — [description]
Message: ACD-[STORY-KEY]: Add/Update [flow-name] flow documentation
```

Use separate commits if both specs and flows were created, or a single commit if only one.

#### Commit 2 — Implementation Code

```
Message: ACD-[STORY-KEY]: [Story summary]
```

Include all code changes.

**If only one category has changes** (e.g., only code, no specs), create a single commit with the appropriate message format.

### Step 3.3 — Ask About Push

Use `AskUserQuestion`:

- **Yes, commit and push** — Commit all changes and push to remote
- **Commit only, don't push** — Save locally but don't push
- **No, continue without committing** — Skip git, move to next story

If committing, execute git operations and show result:

```
Committed:
  [Story commits created and messages shown]

Pushed: ✓ [to origin/current-branch]
```

---

## Stage 4 — Next Story {#exe.4.next-story}

After stage 3 is complete for the current story:

```
Story [STORY-KEY] is complete!
  ✓ All tasks implemented
  ✓ Story marked Done
  ✓ Changes committed and pushed

Next: Move to the next pending story, or wrap up?
```

Use `AskUserQuestion`:

- **Continue to next story** — Implement the next pending story
- **Stop here for now** — Pause execution (will add progress note to epic)
- **Add context before continuing** — Provide additional notes or guidance for the next story

If user wants to add context, gather it. Then repeat the question.

If user continues, return to **Stage 2** with the next pending story.

If user stops, proceed to **Stage 5**.

---

## Stage 5 — Epic Completion {#exe.5.epic-completion}

When all pending stories are complete (or user chooses to stop):

### Step 5.1 — Check if All Work is Done

Fetch the epic again and check if **all child stories and tasks are marked Done**.

- **All done** → Proceed to 5.2
- **Some pending** → Proceed to 5.3 (partial completion)

### Step 5.2 — Finalize Epic (All Work Done)

Transition the epic to **Done** using `transitionJiraIssue`.

Add a final comment to the epic:

```
Epic implementation complete!

Summary:
- Stories implemented: [count]
- Tasks completed: [count]
- Files changed: [count]
- Commits created: [count]

All acceptance criteria met. Epic ready for review and deployment.
```

Present the completion summary:

```
Epic [KEY] — [Title]
Status: ✓ DONE

Implementation Summary:
  Stories:      [X] implemented
  Tasks:        [X] completed
  Files:        [X] changed
  Modules:      [list of affected modules]
  Flow docs:    [list if created]
  Commits:      [count] created

All work is complete!
```

Use `AskUserQuestion`:

- **Done, thanks!** — End the session
- **I want to add final notes** — Add additional context before closing

If user adds notes, add them to the epic comment, then offer to end.

### Step 5.3 — Partial Completion (Some Work Remaining)

If some tasks are still pending:

Transition the epic to a status that reflects partial progress (e.g., **In Progress**, if it's not already).

Add a progress comment to the epic:

```
Execution paused.

Progress:
- Completed:  [stories/tasks listed]
- Pending:    [stories/tasks listed]
- Files:      [count] changed so far
- Commits:    [count] created

Ready to resume execution at any time using: execute epic [KEY]
```

Present the progress summary:

```
Execution Paused

Epic [KEY] — [Title]
Current Status: In Progress

Progress:
  Completed:    [X] stories, [X] tasks
  Pending:      [X] stories, [X] tasks
  Files:        [X] changed
  Commits:      [X] created

You can resume anytime with: execute epic [KEY]
```

Use `AskUserQuestion`:

- **Done for now** — End the session
- **Actually, continue with the next story** — Resume execution (return to Stage 4)
- **I want to add notes** — Add final notes before ending

---

## Summary

**Mode 3 (Execute Epic)** provides a complete implementation workflow:

1. **Load** the epic context, stories, and spec coverage
2. **Execute** each story with spec-first checks, flow detection, and task-by-task implementation
3. **Commit** separately (specs → flow → code) after each story
4. **Ask** whether to continue or stop after each story
5. **Close** the epic with a final summary once all work is done

Each story is a complete cycle: understand → spec → flow → implement → commit → move on.

The user is in control at every step — no surprises, no automation beyond what's confirmed.
