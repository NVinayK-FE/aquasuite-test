---
name: jira-batch-executor
description: >
  Use this skill whenever the user wants to implement several standalone JIRA
  tickets back-to-back. Trigger on phrases like "start jira-batch-executor",
  "execute tickets", "run tickets", "batch execute", or any comma-separated list
  of JIRA ticket keys (e.g., "ACD-1, ACD-2, ACD-3") — even if the user doesn't
  say the word "batch". This skill is a **thin queue iterator**: it fetches the
  tickets, presents the queue, and then hands each ticket to `jira-executor`
  one at a time (routing Bug tickets through `jira-bug-executor` first for the
  analysis phase). It never reimplements understand → dev-flows → spec-first →
  plan → implement → commit → jira-update logic — that all lives in
  `jira-executor`.
---

# JIRA Batch Executor

Execute multiple standalone JIRA tickets in sequence, one at a time. The tickets do not need to belong to the same epic — they are independent work items processed in the order you provide.

This skill is responsible **only** for queue management and delegation. Each ticket is handed off to `jira-executor` (or `jira-bug-executor` first, for Bug tickets) for the full per-ticket flow.

## Per-skill rule — Remember answers for the session {#batch.remember}

Every `AskUserQuestion` prompt raised by this skill MUST follow the Human Confirmation Protocol's **Remember Answer for the Session** rule (see `.claude/skills/jira/_shared/references/human-confirmation-protocol.md` — section [`hcp.8.remember-answer`]).

Concretely:

- The between-tickets "Continue / Pause" prompt (Stage 3) offers a remember variant so the user can lock in "always continue" for the rest of the batch.
- The Stage 1 queue confirmation is typically asked once per batch and so does not need a remember option, but any sub-prompt that repeats within a batch must offer one.
- Destructive operations delegated into `jira-executor` (status transitions, git push) retain their own `hcp.6.destructive-pattern` handling inside that skill — this skill does not remember them on the executor's behalf.
- Cross-project batch warnings (see the "Cross-project spec caveat" below) are informational, not confirmational, and are printed every time they apply.

## Cross-project spec caveat {#batch.cross-project}

`jira-executor`'s Spec-First Check (Step 3) reads `.spec/` **from the current repository only**. When a batch contains ticket keys from more than one JIRA project (e.g. `ACD-1, MD-5`), the specs required for each ticket may live in separate repositories — a single-repo spec scan will not find them.

Detection rule: if the queue contains ticket keys with two or more distinct project prefixes, this is a "cross-project batch".

When detected, print a prominent warning at Stage 1 and ask the user how to proceed:

```
⚠ Cross-project batch detected — projects in this queue: [ACD, MD]

The Spec-First Check reads only the `.spec/` tree of the current repo.
Tickets from other projects may require specs that live elsewhere; those
specs will NOT be scanned or updated by this run.

Options:
  1. Continue anyway — the executor will scan the current repo's `.spec/`
     only; specs for the other project may come back as "missing" and you
     will be prompted to create them here, which is often wrong.
  2. Split the batch — run one batch per project, each in its own repo.
     (Recommended.)
  3. Skip tickets from foreign projects — keep only the tickets that
     match the current repo and drop the rest.
```

Use `AskUserQuestion` to let the user pick. Single-project batches skip this caveat entirely.

## Section IDs

**Prefix:** `batch`

When starting each stage, display the section ID:

```
▶ [batch.N.name] — Stage title
```

| ID | Stage |
|---|---|
| `batch.1.fetch-queue` | Fetch & Present Queue |
| `batch.2.ticket-execution` | Per-Ticket Execution (delegated) |
| `batch.3.progress-next` | Progress & Next Ticket |
| `batch.4.batch-completion` | Batch Completion |

## Principle — Single Execution Path {#batch.principle}

`jira-batch-executor` does NOT reimplement the execution flow. For each ticket in the queue:

- **Task / Subtask / Story** → read `.claude/skills/jira/jira-executor/SKILL.md` and run its flow from Step 1.
- **Bug** → read `.claude/skills/jira/jira-bug-executor/SKILL.md` and run its flow from Step 1; it will hand off to `jira-executor` itself once the diagnosis is confirmed and the ticket description updated.
- **Epic** → do NOT execute an Epic from here. Advise the user to use `execute epic <KEY>` (which routes via `jira-epic-orchestrator`); skip or remove the Epic from the queue on user confirmation.

This keeps all per-ticket logic (spec-first, dev flows, implementation plan, code, In-Progress transition, git commits, and the per-ticket JIRA comment) inside `jira-executor` — no duplication.

## Trigger

```
start jira-batch-executor ACD-1, ACD-2, ACD-3
```

Accepts any number of comma-separated JIRA ticket keys. Tickets can be from different projects (ACD, MD, etc.).

---

## Stage 1 — Fetch & Present Queue {#batch.1.fetch-queue}

Fetch each ticket using `getJiraIssue`. Present the execution queue:

```
▶ [batch.1.fetch-queue] — Execution Queue

Execution Queue: [N] tickets
═══════════════════════════════════════════════════════

 #  | Key       | Type    | Priority | Status      | Summary                    | Routed via
────┼───────────┼─────────┼──────────┼─────────────┼────────────────────────────┼──────────────────
 1  | ACD-1     | Task    | High     | To Do       | Set up project structure   | jira-executor
 2  | ACD-2     | Story   | Medium   | To Do       | User registration API      | jira-executor
 3  | ACD-3     | Bug     | Highest  | In Progress | Fix login redirect loop    | jira-bug-executor → jira-executor

═══════════════════════════════════════════════════════

Execution plan: each ticket will be handed off to `jira-executor` one at
a time. Bugs go through `jira-bug-executor` first (analysis + description
update) and then hand off to `jira-executor` for the actual fix.
```

Validate:

- If any ticket key is invalid, report it and offer to skip or remove it.
- If any ticket is an **Epic**, flag it — Epics cannot be executed here. Offer to remove from the queue or advise `execute epic <KEY>`.
- **Cross-project check:** collect the distinct project prefixes from the queue (the part before the `-` in each key). If two or more distinct prefixes are present, run the `batch.cross-project` caveat warning and prompt **before** the queue-confirmation prompt below. Honour the user's choice (continue / split / skip foreign-project tickets) before proceeding.

Use `AskUserQuestion`:

- **Yes, start executing in this order** — Begin with ticket #1
- **Reorder or remove tickets** — Change the queue
- **Add more tickets** — Append additional ticket keys

Once confirmed, proceed to Stage 2.

---

## Stage 2 — Per-Ticket Execution (delegated) {#batch.2.ticket-execution}

For each ticket in the queue, in order:

### Step 2.1 — Announce the handoff

```
═══════════════════════════════════════════════════════
TICKET [current] of [total]: [KEY] — [Summary]
═══════════════════════════════════════════════════════

▶ [batch.2.ticket-execution] — Handing ticket [KEY] to [skill]

Type:         [Task / Story / Subtask / Bug]
Routed via:   [jira-executor | jira-bug-executor → jira-executor]
```

### Step 2.2 — Delegate

Based on the ticket type:

- **Task / Story / Subtask** — **Read `.claude/skills/jira/jira-executor/SKILL.md` and begin executing its flow from Step 1**, passing the ticket key. `jira-executor` runs the full per-ticket flow:
  - Step 1 — Fetch and present ticket understanding
  - Step 2 — Generate dev flows (code-to-phrase + code-to-sentence)
  - Step 3 — Spec-first check
  - Step 4 — Identify specs to update
  - Step 5 — Present implementation plan
  - Step 6 — Implement the code
  - Step 7 — Transition to In Progress (after code is ready)
  - Step 8 — Git workflow (separate commits for spec / dev-flows / code)
  - Step 9 — Post JIRA update

- **Bug** — **Read `.claude/skills/jira/jira-bug-executor/SKILL.md` and begin executing its flow from Step 1**, passing the ticket key. The bug executor runs its analysis flow, gets user confirmation, writes the analysis back to the ticket description, and then hands off to `jira-executor` for the actual fix.

Do **not** repeat any understanding, spec-first, planning, coding, commit, or JIRA-comment logic inside this skill — that is exactly what the delegated skill exists for.

### Step 2.3 — After delegation returns

When the delegated skill returns, the ticket has been implemented, committed, transitioned, and commented on (per `jira-executor`'s Steps 7–9). Proceed to Stage 3.

---

## Stage 3 — Progress & Next Ticket {#batch.3.progress-next}

Show queue progress:

```
▶ [batch.3.progress-next] — Progress

Queue Progress:
  ✅ [1] ACD-1 — Set up project structure (Done via jira-executor)
  ✅ [2] ACD-2 — User registration API (Done via jira-executor)
  ▶️ [3] ACD-3 — Fix login redirect loop (next — routing via jira-bug-executor)
  ☐  [4] ACD-4 — Add email verification (pending)
```

Use `AskUserQuestion` (remember-variant allowed per `batch.remember` — "Always continue automatically this session" is a valid option):

- **Continue to next ticket** — Hand the next ticket to the appropriate skill (per Stage 2)
- **Stop here for now** — Pause the batch; go to Stage 4 for a partial wrap-up
- **Revisit a completed ticket** — Re-hand a prior ticket to `jira-executor` / `jira-bug-executor`
- **Skip this ticket** — Mark current ticket as skipped and move on
- **Remember: always continue this session** — Don't ask between tickets for the rest of this session

If continuing, return to Stage 2 for the next ticket. If stopping, go to Stage 4.

---

## Stage 4 — Batch Completion {#batch.4.batch-completion}

When all tickets are processed (or the user stops):

```
▶ [batch.4.batch-completion] — Batch Complete

═══════════════════════════════════════════════════════
BATCH COMPLETE
═══════════════════════════════════════════════════════

Tickets Executed: [count]
Tickets Skipped:  [count]
Tickets Pending:  [count]  (if user paused)

Results:
  ✅ ACD-1 — Set up project structure → Done (via jira-executor)
  ✅ ACD-2 — User registration API → Done (via jira-executor)
  ✅ ACD-3 — Fix login redirect loop → Done (via jira-bug-executor → jira-executor)
  ⏭  ACD-4 — Skipped
  ☐  ACD-5 — Pending

Per-ticket implementation details (files changed, specs touched, dev flows,
commits) are recorded on each ticket's own JIRA comment by `jira-executor`.
═══════════════════════════════════════════════════════
```

Use `AskUserQuestion`:

- **Done, thanks!** — End the session
- **Resume remaining tickets** — Go back to Stage 2 for the next pending ticket
- **Add a summary comment to any ticket** — Provide the key and additional notes

---

## Important Principles

- **All per-ticket work runs through `jira-executor`** (via `jira-bug-executor` first for Bugs). This skill never implements code, never runs spec-first, never generates dev flows, never commits, never posts per-ticket JIRA comments.
- **One ticket at a time.** Finish one ticket entirely (via the delegated skill's full flow) before starting the next.
- **Tickets are independent.** These tickets don't need to share a parent epic. They're a queue of standalone work items.
- **Cross-project batches are flagged, not auto-blocked.** The spec scan is single-repo, so the user is warned at Stage 1 before a mixed-project queue starts.
- **Never originate substance.** All substance comes from the tickets, specs, and the delegated skill's own confirmation points.
- **Always confirm before acting.** No delegation without explicit user approval at Stage 1 (queue) and Stage 3 (next). Remember-variants are offered where safe; destructive actions inside the delegated skill keep their own never-remembered confirmation.
- **Respect the queue order.** Process tickets in the order given unless the user reorders or skips.
- **No Epics in the queue.** Epics are executed via `jira-epic-orchestrator`. Flag and remove.
