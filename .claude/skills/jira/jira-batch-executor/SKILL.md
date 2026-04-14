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
     
     Trigger: start jira-batch-executor ACD-1, ACD-2, ACD-3
     
     Each ticket follows the shared Implementation Protocol:
       Understand → Spec-First → Plan → Implement → Git → JIRA update
     
     After each ticket, the full Git Workflow (Level 3) is presented.
     
     Shared protocols used:
       - spec-first-protocol.md     → Check/create specs before coding
       - implementation-protocol.md → Standard execute flow per ticket
       - git-workflow.md            → Full interactive git menu (Level 3)
       - human-confirmation-protocol.md → Enhanced confirmations
       - flow-protocol.md           → Detect and document user flows
     ============================================================ -->

# JIRA Batch Executor

Execute multiple JIRA tickets in sequence, one at a time, with full interactive git control after each ticket. The tickets do not need to belong to the same epic — they are independent work items processed in the order you provide.

**Project:** ACD (Aqua Claude Dev)

**Shared Protocols:** This skill references reusable protocols in `.claude/skills/jira/_shared/references/`. Read the relevant protocol file when a step says "follow [protocol-name]".

## Section IDs

**Prefix:** `batch`

When starting each stage, display the section ID:

```
▶ [batch.N.name] — Stage title
```

| ID | Stage |
|---|---|
| `batch.1.fetch-queue` | Fetch & Present Queue |
| `batch.2.ticket-execution` | Ticket-by-Ticket Execution |
| `batch.3.git-workflow` | Git Workflow (per Ticket) |
| `batch.4.jira-update-next` | JIRA Update & Next Ticket |
| `batch.5.batch-completion` | Batch Completion |

## How It Relates to Other Skills

| Skill | Input | Scope | Git control |
|---|---|---|---|
| `jira-executor` | 1 ticket key | Single ticket | Basic (commit + push) |
| `jira-batch-executor` | Comma-separated keys | Multiple independent tickets | Full git menu after each |
| `jira-epic-orchestrator` | Epic key or feature idea | Epic hierarchy (stories → tasks) | Standard (commit per story) |

---

## Trigger

```
start jira-batch-executor ACD-1, ACD-2, ACD-3
```

Accepts any number of comma-separated JIRA ticket keys. Tickets can be from different projects (ACD, MD, etc.).

---

## Stage 1 — Fetch & Present Queue {#batch.1.fetch-queue}

Fetch all provided tickets using `getJiraIssue`. Present the execution queue:

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

If any ticket key is invalid, report it and mark for skipping.

Use `AskUserQuestion` following the **human-confirmation-protocol.md**:

- **Yes, start executing in this order** — Begin with ticket #1
- **No, let me reorder or remove tickets** — I want to change the queue
- **I want to add more tickets** — Add additional ticket keys to the queue

---

## Stage 2 — Ticket-by-Ticket Execution {#batch.2.ticket-execution}

For each ticket, show a progress header:

```
═══════════════════════════════════════════════════════
TICKET [current] of [total]: [KEY] — [Summary]
═══════════════════════════════════════════════════════
```

Then **follow the Implementation Protocol** in `.claude/skills/jira/_shared/references/implementation-protocol.md` exactly as defined. This encompasses:

1. **Present understanding** — Fetch ticket, rephrase requirements, confirm with enhanced options (correct / partially correct / add more context)
2. **Transition to In Progress** — Move ticket status  
3. **Run Spec-First Protocol** — Follow `.claude/skills/jira/_shared/references/spec-first-protocol.md` to verify all required specs exist. Create any missing specs before proceeding.
3b. **Run Flow Check** — Follow `.claude/skills/jira/_shared/references/flow-protocol.md`. After specs are confirmed, detect if the ticket involves a user flow. If yes, walk through the flow protocol to create/update flow documentation. Flow files will be committed separately in the git workflow stage.
4. **Identify relevant spec files** — Scan .spec/ files, present relevant ones, confirm selection
5. **Present implementation plan** — Show file changes and code snippets, confirm with enhanced options (proceed / adjust / add more details)
6. **Implement** — Execute code changes, show acceptance criteria check, confirm satisfaction

All confirmations in this stage must follow **human-confirmation-protocol.md**, which includes the "add more details" option for every confirmation point.

---

## Stage 3 — Git Workflow (per Ticket) {#batch.3.git-workflow}

After each ticket's implementation, categorize changed files into:
- Spec files (`.spec/`)
- Flow files (`.flows/`)
- Implementation code (all other files)

Commit each category separately in that order with these commit messages:
- Spec files: `[KEY]: Add/Update spec — [description]`
- Flow files: `[KEY]: Add/Update [flow-name] flow documentation`  
- Implementation: `[KEY]: [Summary]`

After committing each category, **follow the Git Workflow Protocol (Level 3 — Full)** in `.claude/skills/jira/_shared/references/git-workflow.md`.

The full interactive git menu loops until the user chooses to move on:

| Option | What it does |
|---|---|
| Commit changes | Stage + commit with auto-generated or custom message |
| Push to remote | Push current branch to origin |
| Create new branch | Create from current, main, or specified base branch |
| Switch branch | Switch with stash/commit warning for uncommitted changes |
| Merge branch | Merge another branch into current, with conflict resolution |
| Pull latest | Pull from remote with conflict handling |
| Create Pull Request | Create PR via gh CLI with title, target branch, description |
| Done, move on | Exit git menu, proceed to next ticket |

Refer to the shared protocol file for the complete decision tree and handling of each option.

---

## Stage 4 — JIRA Update & Next Ticket {#batch.4.jira-update-next}

### Update JIRA

1. **Add a comment** to the ticket using `addCommentToJiraIssue` with implementation summary, files changed, and git operations performed.

2. Use `AskUserQuestion` following **human-confirmation-protocol.md**:
   - **Mark as Done** — Transition ticket to Done
   - **Keep In Progress** — Leave in current status (e.g., waiting for review)
   - **Custom status** — I'll specify which status to set
   - **Add more details** — I want to provide additional context before deciding

### Progress & Next Ticket

Show queue progress:

```
Queue Progress:
  ✅ [1] ACD-1 — Set up project structure (Done)
  ✅ [2] ACD-2 — User registration API (Done)
  ▶️ [3] ACD-3 — Fix login redirect loop (next)
  ☐  [4] ACD-4 — Add email verification (pending)
```

Use `AskUserQuestion` following **human-confirmation-protocol.md**:

- **Continue to next ticket** — Start executing next ticket
- **Stop here for now** — Pause the batch
- **I want to revisit a completed ticket** — Go back and check something
- **Add more details** — I have additional context about this batch

If stopping, present summary of all work done.
If continuing, go back to Stage 2 for the next ticket.

---

## Stage 5 — Batch Completion {#batch.5.batch-completion}

When all tickets are processed:

```
═══════════════════════════════════════════════════════
BATCH COMPLETE
═══════════════════════════════════════════════════════

Tickets Executed: [count]
Tickets Skipped:  [count]

Results:
  ✅ ACD-1 — Set up project structure → Done
  ✅ ACD-2 — User registration API → Done
  ✅ ACD-3 — Fix login redirect loop → Done

Git Summary:
  Commits: [count]
  Branches created: [list]
  PRs created: [list with URLs]

Total Files Changed: [count]
═══════════════════════════════════════════════════════
```

---

## Important Principles

- **One ticket at a time.** Complete one fully (implement + git + JIRA update) before starting the next.
- **Spec-first.** Always check for required specs before coding. Create missing specs before implementation.
- **Full git control.** The Level 3 git menu gives complete control over branching, commits, merges, and PRs after every ticket.
- **Tickets are independent.** These tickets don't need to share a parent epic. They're a queue of standalone work items.
- **Never originate substance.** All implementation decisions come from ticket requirements and spec files.
- **Always confirm before acting.** No code, git operations, or JIRA transitions without explicit user approval. Always offer "add more details" as an option (per human-confirmation-protocol.md).
- **Respect the queue order.** Process tickets in the order given unless the user reorders or skips.
