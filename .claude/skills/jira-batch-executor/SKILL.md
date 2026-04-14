---
name: jira-batch-executor
description: >
  MANDATORY TRIGGERS: "start jira-batch-executor", "execute tickets",
  "run tickets", "batch execute", or any comma-separated list of JIRA ticket
  keys (e.g., "ACD-1, ACD-2, ACD-3"). Use this skill whenever the user wants
  to implement multiple standalone JIRA tickets in sequence, one after another,
  with full git control (commit, push, branch, merge, pull, PR) after each ticket.
  This is different from jira-executor (single ticket) and jira-epic-orchestrator
  (epic-level). This skill handles a queue of independent tickets processed
  in order with interactive git workflow between each.
---

<!-- ============================================================
     JIRA BATCH EXECUTOR
     ============================================================
     Execute multiple JIRA tickets in sequence, one at a time.
     
     Trigger: `start jira-batch-executor ACD-1, ACD-2, ACD-3`
              or `execute tickets ACD-1, ACD-2, ACD-3`
     
     Each ticket follows the same disciplined flow as jira-executor:
       Fetch → Understand → Confirm → Scan specs → Plan → Implement → Git → JIRA update
     
     After each ticket, a full interactive git menu is presented:
       Commit, push, create branch, switch branch, merge, pull, create PR
     
     Key difference from other skills:
       - jira-executor       → 1 ticket at a time
       - jira-batch-executor → N tickets in sequence with git control
       - jira-epic-orchestrator → Epic-level (stories + tasks as a hierarchy)
     
     These tickets do NOT need to belong to the same epic.
     They are processed independently, in the order given.
     ============================================================ -->

# JIRA Batch Executor

Execute multiple JIRA tickets in sequence, one at a time, with full interactive git control after each ticket. The tickets do not need to belong to the same epic — they are independent work items processed in the order you provide.

**Project:** ACD (Aqua Claude Dev)

## How It Relates to Other Skills

| Skill | Input | Scope | Git control |
|---|---|---|---|
| `jira-executor` | 1 ticket key | Single ticket | Commit + push only |
| `jira-batch-executor` | Comma-separated ticket keys | Multiple independent tickets | Full git menu after each |
| `jira-epic-orchestrator` | Epic key or feature idea | Epic hierarchy (stories → tasks) | Commit per story |

---

## Trigger

```
start jira-batch-executor ACD-1, ACD-2, ACD-3
```

or

```
execute tickets ACD-1, ACD-2, ACD-3
```

Accepts any number of comma-separated JIRA ticket keys (e.g., `ACD-1, ACD-2, MD-5, ACD-10`). Tickets can be from different projects (ACD, MD, etc.).

---

<!-- ============================================================
     STAGE 1 — FETCH & PRESENT QUEUE
     ============================================================
     Fetch all tickets upfront and present the execution queue.
     Let the user review the order, reorder, or remove tickets
     before execution begins.
     ============================================================ -->

## Stage 1 — Fetch & Present Queue

Fetch all provided tickets using `getJiraIssue` for each key. Present the execution queue:

```
Execution Queue: [N] tickets
═══════════════════════════════════════════════════════

 #  | Key       | Type    | Priority | Status      | Summary
────┼───────────┼─────────┼──────────┼─────────────┼──────────────────────────
 1  | ACD-1     | Task    | High     | To Do       | Set up project structure
 2  | ACD-2     | Story   | Medium   | To Do       | User registration API
 3  | ACD-3     | Bug     | Highest  | In Progress | Fix login redirect loop

═══════════════════════════════════════════════════════
```

If any ticket key is invalid or cannot be fetched, report it:

```
⚠️  Could not fetch: ACD-99 (not found), ACD-100 (permission denied)
    These will be skipped.
```

Use `AskUserQuestion` to confirm:

- **Yes, start executing in this order** — Begin with ticket #1
- **No, let me reorder or remove tickets** — I want to change the queue

If reordering, ask for the new order or which tickets to remove, then re-present the queue. Repeat until confirmed.

---

<!-- ============================================================
     STAGE 2 — TICKET-BY-TICKET EXECUTION
     ============================================================
     For each ticket in the queue, follow the same disciplined flow
     as jira-executor:
       a. Present understanding & confirm
       b. Transition to In Progress
       c. Scan relevant spec files
       d. Present implementation plan with code snippets
       e. Implement after confirmation
       f. Show completion summary
     
     This mirrors jira-executor Steps 1–5 exactly.
     ============================================================ -->

## Stage 2 — Ticket-by-Ticket Execution

For each ticket in the queue, execute the following steps. Show a progress header before each ticket:

```
═══════════════════════════════════════════════════════
TICKET [current] of [total]: [KEY] — [Summary]
═══════════════════════════════════════════════════════
```

### Step 2a — Present Understanding

<!-- Like jira-executor Step 1: fetch, read, rephrase, confirm -->

Fetch the full ticket details. Present your understanding:

```
Ticket:      [KEY] — [Summary]
Type:        [Issue Type]
Priority:    [Priority]
Status:      [Current Status]

My Understanding:
[Rephrase the ticket requirements — what needs to be done, why,
and what the expected outcome is]

Acceptance Criteria:
- [criterion 1]
- [criterion 2]
- ...
```

Use `AskUserQuestion`:

- **Yes, that's correct** — Your understanding is accurate
- **No, let me clarify** — I need to add context or correct something
- **Skip this ticket** — Move to the next ticket in the queue

If clarifying, collect input, update understanding, re-present. Repeat until confirmed.

### Step 2b — Transition to In Progress

<!-- Like jira-executor Step 2 -->

Transition the ticket to **In Progress** using `transitionJiraIssue`. Inform the user.

### Step 2c — Scan Relevant Spec Files

<!-- Like jira-executor Step 3 -->

Scan all `.spec` markdown files in the project. List relevant files with their sections:

```
Relevant Spec Files:

1. [file path]
   - [section-id] — [section title]
   - [section-id] — [section title]

2. [file path]
   - [section-id] — [section title]
```

If no spec files are relevant, state that clearly.

Use `AskUserQuestion`:

- **Yes, this is correct** — These are the right spec files
- **No, let me adjust** — I want to add or remove sections
- **Guide me** — Show all available spec files so I can pick

### Step 2d — Present Implementation Plan

<!-- Like jira-executor Step 4 -->

Based on the ticket and confirmed specs, present a concrete implementation plan:

```
Implementation Plan:

1. [File path — new/modified]
   Action: [create / modify / delete]
   Impact: [what this change does]
   
   Code snippet:
   [key code changes — not full file, just the relevant parts]

2. [File path — new/modified]
   Action: [create / modify / delete]
   Impact: [what this change does]
   
   Code snippet:
   [key code changes]
```

Present only what the ticket and specs require — nothing more. If you believe something additional is needed, list it separately as a suggestion.

Use `AskUserQuestion`:

- **Yes, proceed** — Looks good, start implementing
- **No, let me adjust** — I want to change the approach

### Step 2e — Implement

<!-- Like jira-executor Step 5 -->

Execute the code changes exactly as confirmed. After completion:

```
Implementation Complete:

Files changed:
- [file path] — [what was done]
- [file path] — [what was done]

Acceptance Criteria Check:
✅ [criterion 1] — [how it was met]
✅ [criterion 2] — [how it was met]
```

---

<!-- ============================================================
     STAGE 3 — GIT WORKFLOW (after each ticket)
     ============================================================
     Full interactive git menu after every ticket completes.
     The user picks which git operations to perform.
     Multiple operations can be chained (e.g., create branch,
     commit, push, then create PR).
     
     This is the key differentiator from jira-executor which
     only offers commit + push.
     ============================================================ -->

## Stage 3 — Git Workflow (per Ticket)

After each ticket's implementation is complete, present the full interactive git menu. The user can perform multiple operations in sequence — the menu repeats until they choose to move on.

### Git Menu

Use `AskUserQuestion`:

- **Commit changes** — Stage and commit the changes for this ticket
- **Push to remote** — Push the current branch to remote
- **Create new branch** — Create and switch to a new branch
- **Switch branch** — Switch to an existing branch
- **Merge branch** — Merge another branch into the current branch
- **Pull latest** — Pull latest changes from remote
- **Create Pull Request** — Create a PR on GitHub for the current branch
- **Done, move to next ticket** — Skip remaining git operations and continue

#### If **Commit changes**:

Ask for commit message preference using `AskUserQuestion`:

- **Auto-generate** — Use format: `[TICKET-KEY]: [Summary]` (e.g., `ACD-1: Set up project structure`)
- **Let me write it** — I'll provide the commit message

Stage all changes for this ticket and commit. Show the result:

```
Committed: [commit hash]
Message:   [commit message]
Branch:    [current branch]
Files:     [count] changed
```

**Return to Git Menu** — the user may want to push, create PR, etc.

#### If **Push to remote**:

Push the current branch. Show the result:

```
Pushed: [branch] → origin/[branch]
```

**Return to Git Menu.**

#### If **Create new branch**:

Ask using plain text: "What should the new branch be named?"

Then ask using `AskUserQuestion`:

- **Branch from current branch** — Create from where I am now ([current branch])
- **Branch from main** — Create from the main/master branch
- **Branch from another branch** — I'll specify the base branch

Create the branch, switch to it, and confirm:

```
Created and switched to: [new-branch]
Based on: [base-branch]
```

**Return to Git Menu.**

#### If **Switch branch**:

List available local branches and ask which to switch to. If there are uncommitted changes, warn:

```
⚠️  You have uncommitted changes. Switching branches may cause conflicts.
```

Use `AskUserQuestion`:

- **Stash and switch** — Stash my changes, switch, then I'll unstash later
- **Commit first, then switch** — Commit current changes before switching
- **Cancel** — Stay on current branch

After switching, confirm:

```
Switched to: [branch-name]
```

**Return to Git Menu.**

#### If **Merge branch**:

Ask using plain text: "Which branch do you want to merge into the current branch ([current branch])?"

Perform the merge. If there are conflicts:

```
⚠️  Merge conflicts detected in:
- [file path]
- [file path]
```

Use `AskUserQuestion`:

- **Help me resolve** — Show the conflicts and help me fix them
- **Abort merge** — Cancel the merge and return to the previous state

If no conflicts:

```
Merged: [source-branch] → [current-branch]
Commits merged: [count]
```

**Return to Git Menu.**

#### If **Pull latest**:

Pull from remote for the current branch. Show result:

```
Pulled: origin/[branch] → [branch]
[count] new commits
```

If there are conflicts, handle the same way as merge conflicts.

**Return to Git Menu.**

#### If **Create Pull Request**:

Ask these one at a time:

1. "What's the PR title?" (default suggestion: `[TICKET-KEY]: [Summary]`)
2. "What's the target branch?" using `AskUserQuestion`:
   - **main** — Merge into main branch
   - **develop** — Merge into develop branch
   - **Other** — I'll specify the target branch
3. "Any additional description for the PR?" (optional, plain text)

Create the PR using `gh pr create`. Show the result:

```
Pull Request Created:
  Title:   [PR title]
  Branch:  [source] → [target]
  URL:     [PR URL]
  Ticket:  [TICKET-KEY]
```

**Return to Git Menu.**

#### If **Done, move to next ticket**:

Exit the git menu and proceed to Stage 4.

---

<!-- ============================================================
     STAGE 4 — JIRA UPDATE & NEXT TICKET
     ============================================================
     After git operations, update the JIRA ticket and move on.
     ============================================================ -->

## Stage 4 — JIRA Update & Next Ticket

### Update JIRA

1. **Add a comment** to the ticket using `addCommentToJiraIssue` with the implementation summary, files changed, and any git operations performed (branch, PR link, etc.).

2. Ask using `AskUserQuestion`:
   - **Mark as Done** — Transition ticket to Done
   - **Keep In Progress** — Leave it in current status (e.g., waiting for code review)
   - **Custom status** — I'll tell you which status to set

### Progress & Next Ticket

Show queue progress:

```
Queue Progress:
  ✅ [1] ACD-1 — Set up project structure (Done)
  ✅ [2] ACD-2 — User registration API (Done)
  ▶️ [3] ACD-3 — Fix login redirect loop (next)
  ☐  [4] ACD-4 — Add email verification (pending)
```

Use `AskUserQuestion`:

- **Continue to next ticket** — Start executing [NEXT-KEY]
- **Stop here for now** — Pause the batch

If stopping, present a final summary of all work done so far.

If continuing, go back to **Stage 2** for the next ticket.

---

<!-- ============================================================
     STAGE 5 — BATCH COMPLETION
     ============================================================
     When all tickets are processed, present a final summary.
     ============================================================ -->

## Stage 5 — Batch Completion

When all tickets in the queue have been processed:

```
═══════════════════════════════════════════════════════
BATCH COMPLETE
═══════════════════════════════════════════════════════

Tickets Executed: [count]
Tickets Skipped:  [count] (if any)

Results:
  ✅ ACD-1 — Set up project structure → Done
  ✅ ACD-2 — User registration API → Done
  ✅ ACD-3 — Fix login redirect loop → Done
  ⏭️  ACD-4 — Add email verification → Skipped

Git Summary:
  Commits: [count]
  Branches created: [list]
  PRs created: [list with URLs]

Total Files Changed: [count]
═══════════════════════════════════════════════════════
```

---

## Important Principles

- **One ticket at a time.** Never implement multiple tickets simultaneously. Complete one fully (implement + git + JIRA update) before starting the next.
- **Full git control.** The git menu is the key differentiator — users get complete control over branching strategy, commits, merges, and PRs after every single ticket.
- **Tickets are independent.** Unlike the epic orchestrator, these tickets don't need to share a parent or belong to the same epic. They're a queue of standalone work items.
- **Never originate substance.** All implementation decisions come from the ticket requirements and spec files. You propose, the user decides.
- **Always confirm before acting.** No code is written, no git operations are performed, and no JIRA transitions happen without explicit user approval.
- **Respect the queue order.** Process tickets in the order given unless the user explicitly reorders or skips.
- **How this relates to other skills:** `jira-executor` handles 1 ticket with basic git. `jira-batch-executor` handles N tickets with full git. `jira-epic-orchestrator` handles epic hierarchies. All three follow the same disciplined flow: understand → confirm → implement → update.
