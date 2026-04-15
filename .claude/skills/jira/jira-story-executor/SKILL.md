---
name: jira-story-executor
description: >
  Use this skill whenever the user wants to execute a single JIRA Story and all
  of its subtasks in sequence. Trigger on "start jira-story-executor",
  "execute story", "implement story", "work on story", or commands like
  "execute story ACD-8". It is the story-level counterpart to the epic
  orchestrator's execute mode: spec-first runs once at the story level, each
  subtask is implemented in turn, and git control kicks in after the story is
  complete. Prefer this over jira-executor (which handles a single Task/Subtask
  in isolation) and over jira-batch-executor (which handles multiple unrelated
  tickets).
---

<!-- JIRA Story Executor — 5 stages (load → spec/flow check → subtask execution → git workflow → completion).
     Spec-first runs once at the story level; each subtask is implemented in sequence; Level 3 git menu
     after the story is complete. Uses shared protocols: spec-first, flow, implementation, human-confirmation,
     git-workflow (Level 3). See `executor-comparison.md` for how this compares to the other executors. -->

# JIRA Story Executor

Execute a single JIRA Story and all its subtasks in sequence. The spec-first check runs ONCE at the story level (not per subtask), then each subtask is implemented one by one with user confirmation at every step.

**Project:** ACD (Aqua Claude Dev)

**Shared Protocols:** This skill references reusable protocols in `.claude/skills/jira/_shared/references/`. Read the relevant protocol file when a step says "follow [protocol-name]".

## Section IDs

**Prefix:** `story`

When starting each stage, display the section ID:

```
▶ [story.N.name] — Stage title
```

| ID | Stage |
|---|---|
| `story.1.load-context` | Load Story Context |
| `story.2.spec-flow-check` | Spec-First & Flow Check |
| `story.3.subtask-execution` | Subtask-by-Subtask Execution |
| `story.4.git-workflow` | Git Workflow |
| `story.5.story-completion` | Story Completion |

## How It Relates to Other Skills

See `.claude/skills/jira/_shared/references/executor-comparison.md` for the full comparison table and the distinctions between story-executor, batch-executor, and the epic orchestrator's execute mode. In short: this skill is for a single Story with its subtasks processed as one cohesive unit — spec-first runs once at the story level and Level 3 git control kicks in after all subtasks are done.

---

## Trigger

```
execute story ACD-8
start jira-story-executor ACD-8
```

Accepts a single JIRA Story key. The story must have subtasks.

---

## Stage 1 — Load Story Context {#story.1.load-context}

### Step 1.1 — Fetch Story and Subtasks

Use `getJiraIssue` to fetch the story by the provided key. Then fetch all child subtasks using `searchJiraIssuesUsingJql`:

```
JQL: parent = <STORY-KEY> ORDER BY created ASC
```

Extract:
- Story key, title, description, acceptance criteria
- Parent epic key and title (if any)
- All child subtasks (key, summary, status, priority)
- Story current status

### Step 1.2 — Load Spec File

Check if the story belongs to an epic. If so, look for the epic's spec file:

- Search `.spec/epics/<name>.md` for the parent epic
- Also check root `CLAUDE.md` Registered Specs for the epic ID

If found, read the spec file and extract the section relevant to this story (task details, technical specs, acceptance criteria).

If not found, note it — the spec-first check in Stage 2 will catch any gaps.

### Step 1.3 — Filter to Pending Subtasks

From all child subtasks, identify which are pending (status != Done).

Present the overview:

```
▶ [story.1.load-context] — Load Story Context

═══════════════════════════════════════════════════════
STORY: [KEY] — [Title]
═══════════════════════════════════════════════════════

Type:        Story
Priority:    [Priority]
Status:      [Current Status]
Parent Epic: [EPIC-KEY] — [Epic Title] (or "None")

Acceptance Criteria:
- [criterion 1]
- [criterion 2]

Spec File:   [Found / Not Found]
  Location:  [path or "N/A"]

Subtasks: [total] total, [pending] pending, [done] already done

 #  | Key       | Priority | Status  | Summary
────┼───────────┼──────────┼─────────┼──────────────────────────
 1  | ACD-18    | Medium   | Open    | Initialize Next.js project
 2  | ACD-19    | Medium   | Open    | Configure TypeScript
 ...

═══════════════════════════════════════════════════════
```

If any subtasks are already Done, mark them with ✅ in the list.

Use `AskUserQuestion` following **human-confirmation-protocol.md**:

- **Yes, let's start** — Begin with spec-first check then execute subtasks
- **No, I want to review first** — Show me more details about each subtask
- **Add more context first** — I have additional notes about this story

If user wants to review, show each pending subtask with its full description. Then repeat the question.

---

## Stage 2 — Spec-First & Flow Check (Story Level) {#story.2.spec-flow-check}

This stage runs ONCE for the entire story — covering ALL subtasks. Individual subtasks do NOT re-run spec-first.

### Step 2.1 — Run Spec-First Protocol

**Read and follow `.claude/skills/jira/_shared/references/spec-first-protocol.md`.**

Analyze the story AND all its subtask descriptions to identify every spec needed:

1. **Identify Required Specs** — Check all categories (naming, architecture, config, API, DB, business rules) across ALL subtasks
2. **Deep Scan** — Read each spec file, check section IDs, assess completeness
3. **Gap Report** — Present all missing files, missing sections, and incomplete sections
4. **Create/Update Specs** — Interactively create missing specs with human confirmation
5. **Final Confirmation** — Hard gate: all specs must be confirmed before any subtask is implemented

**Important:** Because this covers ALL subtasks in the story, the spec analysis should be comprehensive. For example, if Subtask 1 creates folders and Subtask 3 creates config files, the spec check should catch naming conventions AND config specs in this single pass.

### Step 2.2 — Run Flow Detection

**Read and follow `.claude/skills/jira/_shared/references/flow-protocol.md` (Step 1 — Flow Detection).**

Analyze the story and ALL subtask descriptions for user-facing flows:

```
▶ [story.2.spec-flow-check] — Flow Detection

Analyzing story [KEY] and all [N] subtasks for user flows...

[Summary: "This story involves a [flow type]" or "No user flows detected"]
```

Use `AskUserQuestion`:

- **Yes, create flow documentation** — Proceed with flow protocol
- **No, skip flow docs** — Continue without flow documentation
- **I want to add more context about the flow** — Provide additional details first

If creating flow docs, follow the full flow protocol (Steps 2-7). Flow files will be committed separately in Stage 4.

### Step 2.3 — Transition Story to In Progress

After specs and flow check are complete, transition the story to **Claude In Progress** (or **In Progress**) using `transitionJiraIssue`.

```
[STORY-KEY]: Now Claude In Progress
All specs confirmed. Ready to execute [N] subtasks.
```

---

## Stage 3 — Subtask-by-Subtask Execution {#story.3.subtask-execution}

For each pending subtask (in order):

### Step 3.1 — Present Subtask Understanding

Show a progress header:

```
═══════════════════════════════════════════════════════
SUBTASK [current] of [total]: [KEY] — [Summary]
═══════════════════════════════════════════════════════
```

Fetch the subtask and present understanding:

```
Subtask:     [KEY] — [Summary]
Type:        Subtask
Priority:    [Priority]
Status:      [Current Status]

My Understanding:
[Rephrase requirements — what, why, expected outcome]

From Story Spec:
[Technical specs from the epic spec file relevant to this subtask]

Acceptance Criteria:
- [criterion 1]
- [criterion 2]
```

Confirm using **Human Confirmation Protocol**:

- **Yes, that's correct** — Proceed with implementation
- **Partially correct, let me clarify** — Some parts need correction
- **I want to add more context** — Additional details before implementing
- **Skip this subtask** — Move to next subtask without implementing

### Step 3.2 — Transition Subtask to In Progress

Use `transitionJiraIssue` to move the subtask to **Claude In Progress**.

### Step 3.3 — Present Implementation Plan

**Follow `.claude/skills/jira/_shared/references/implementation-protocol.md` Step 4 (Present Implementation Plan).**

Show file changes and code snippets based on ticket requirements and confirmed specs:

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

Confirm:

- **Yes, proceed with this** — Looks good, go ahead
- **No, let me adjust** — I want to change the approach
- **Looks good but I want to add more details** — Additional constraints before proceeding

### Step 3.4 — Implement

**Follow `.claude/skills/jira/_shared/references/implementation-protocol.md` Step 5 (Implement).**

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

### Step 3.5 — Update Subtask JIRA

Add a comment to the subtask with:
- Implementation summary
- Files changed
- Acceptance criteria status

Use `AskUserQuestion`:

- **Mark as Done** — Transition subtask to Done
- **Keep In Progress** — Leave in current status (e.g., waiting for review)
- **Add more details** — Additional context before deciding

### Step 3.6 — Progress Check & Next Subtask

Show progress:

```
Subtask Progress:
  ✅ [1] ACD-18 — Initialize Next.js project (Done)
  ✅ [2] ACD-19 — Configure TypeScript (Done)
  ▶️ [3] ACD-20 — Configure ESLint & Prettier (next)
  ☐  [4] ACD-21 — Install core dependencies (pending)
  ☐  [5] ACD-22 — Set up base folder structure (pending)
```

Use `AskUserQuestion`:

- **Continue to next subtask** — Start executing next subtask
- **Stop here for now** — Pause execution
- **I want to revisit a completed subtask** — Go back and check something
- **Add more details** — Additional context before continuing

If stopping, proceed to Stage 4 (git) and then Stage 5 (completion with partial progress).

If continuing, return to Step 3.1 for the next subtask.

---

## Stage 4 — Git Workflow {#story.4.git-workflow}

After all subtasks are complete (or user stops), categorize changed files:

### Step 4.1 — Categorize Changed Files

```
▶ [story.4.git-workflow] — Git Workflow

Changed Files Summary:

Spec files (.spec/):
- [file] — [created/updated]

Flow files (.flows/):
- [file] — [created/updated]

Implementation code:
- [file] — [created/modified]
```

### Step 4.2 — Separate Commits

Commit each category separately in order:

1. **Spec files** (if any): `[STORY-KEY]: Add/Update spec — [description]`
2. **Flow files** (if any): `[STORY-KEY]: Add/Update [flow-name] flow documentation`
3. **Implementation code**: `[STORY-KEY]: [Story summary]`

For each commit, auto-generate the message and present for confirmation.

### Step 4.3 — Full Git Menu (Level 3)

**Read and follow `.claude/skills/jira/_shared/references/git-workflow.md` Level 3 — Full.**

Present the full interactive git menu that **loops** until the user chooses to move on:

| Option | What it does |
|---|---|
| Commit changes | Stage + commit with auto-generated or custom message |
| Push to remote | Push current branch to origin |
| Create new branch | Create from current, main, or specified base branch |
| Switch branch | Switch with stash/commit warning for uncommitted changes |
| Merge branch | Merge another branch into current, with conflict resolution |
| Pull latest | Pull from remote with conflict handling |
| Create Pull Request | Create PR via `gh` CLI with title, target branch, description |
| Done, move on | Exit git menu, proceed to story completion |

Refer to the shared protocol file for the complete decision tree and handling of each option.

---

## Stage 5 — Story Completion {#story.5.story-completion}

### Step 5.1 — Check Completion Status

Check if ALL subtasks are marked Done.

- **All done** → Proceed to 5.2 (full completion)
- **Some pending** → Proceed to 5.3 (partial completion)

### Step 5.2 — Full Story Completion

Transition the story to **Done** using `transitionJiraIssue`.

Add a final comment to the story:

```
All subtasks completed.

Summary:
- Subtasks implemented: [count]
- Files changed: [count]
- Specs created/updated: [count]
- Flows created: [count]
- Commits: [count]

Subtask results:
  ✅ [KEY] — [summary]
  ✅ [KEY] — [summary]
  ...

Story ready for review.
```

Present the completion summary:

```
▶ [story.5.story-completion] — Story Complete

═══════════════════════════════════════════════════════
STORY COMPLETE
═══════════════════════════════════════════════════════

Story:          [KEY] — [Title]
Status:         ✓ Done
Parent Epic:    [EPIC-KEY] — [Title]

Subtasks:       [N] implemented
Files changed:  [N]
Specs:          [N] created/updated
Flows:          [N] created
Commits:        [N]
PRs created:    [list with URLs, if any]

═══════════════════════════════════════════════════════
```

Use `AskUserQuestion`:

- **Done, thanks!** — End the session
- **I want to add final notes** — Add additional context to the story comment
- **Execute the next story** — Pick up the next story in the epic (provide key)

### Step 5.3 — Partial Completion

If some subtasks are still pending (user stopped early):

Keep the story in its current status (Claude In Progress).

Add a progress comment to the story:

```
Execution paused.

Progress:
- Completed: [subtasks listed]
- Pending:   [subtasks listed]
- Files changed: [count] so far
- Commits: [count]

Resume with: execute story [STORY-KEY]
```

Present the progress summary:

```
▶ [story.5.story-completion] — Execution Paused

═══════════════════════════════════════════════════════
EXECUTION PAUSED
═══════════════════════════════════════════════════════

Story:       [KEY] — [Title]
Status:      Claude In Progress

Completed:   [X] subtasks
Pending:     [X] subtasks
Files:       [X] changed
Commits:     [X] created

Resume anytime with: execute story [KEY]

═══════════════════════════════════════════════════════
```

Use `AskUserQuestion`:

- **Done for now** — End the session
- **Actually, continue** — Resume execution (return to Stage 3)
- **Add notes** — Additional context before ending

---

## Important Principles

- **Spec-first runs ONCE per story.** This is the key efficiency gain over batch-executor. All subtask specs are identified and resolved in a single pass before any coding begins.
- **One subtask at a time.** Complete one fully (implement + JIRA update) before starting the next.
- **Full git control.** Level 3 git menu (same as batch-executor) gives complete control over branching, commits, merges, and PRs after the story.
- **Never originate substance.** All implementation decisions come from ticket requirements and spec files.
- **Always confirm before acting.** No code, git operations, or JIRA transitions without explicit user approval.
- **Subtask order matters.** Process subtasks in creation order (ascending) unless the user reorders.
- **Story is the unit.** Git commits are per-story (not per-subtask), keeping the commit history clean and reviewable.
