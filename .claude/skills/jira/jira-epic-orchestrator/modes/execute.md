<!-- ============================================================
     MODE 3 — EXECUTE EPIC
     ============================================================
     Trigger: execute epic <EPIC-KEY>
     Example: execute epic ACD-5

     Purpose: Execute an existing epic by iterating pending child
     tickets and DELEGATING each one to the appropriate executor.

     This mode is a THIN ORCHESTRATOR. It never reimplements
     understand → dev-flows → spec-first → plan → implement →
     commit → jira-update. All of that lives in `jira-executor`.

     Per-child routing:
       - Story   → read jira-story-executor/SKILL.md and hand off
                   (which itself delegates each subtask to
                   jira-executor).
       - Task    → read jira-executor/SKILL.md and hand off directly.
       - Bug     → read jira-bug-executor/SKILL.md and hand off
                   (analysis → description update → jira-executor).
       - Subtask (rare at epic level) → same as Task.

     This mode implements ONLY pending/unfinished work. Already
     Done children are skipped.

     4 Stages:
       1. Load Epic Context — Fetch epic, children, pending list
       2. Child-by-Child Execution — Delegate to the right skill
       3. Next Child — Continue / stop / revisit
       4. Epic Completion — Close the epic and summarize
     ============================================================ -->

# Mode 3 — Execute Epic

**Trigger:** `execute epic <EPIC-KEY>` (e.g., `execute epic ACD-5`)

Execute an existing JIRA epic by iterating its pending child tickets and **delegating each one** to the correct downstream executor. This mode only handles epic-level orchestration — every per-ticket step (understanding, dev flows, spec-first, planning, coding, transitions, commits, JIRA updates) is handled by `jira-executor` (directly for Tasks, via `jira-story-executor` for Stories, via `jira-bug-executor` for Bugs).

**Key Principles:**

- Only implement pending/unfinished tickets — skip Done items.
- All per-ticket execution runs through `jira-executor` — no duplication here.
- Stories are handed to `jira-story-executor`, which iterates its own subtasks and delegates each subtask to `jira-executor`.
- Bugs are handed to `jira-bug-executor`, which runs the analysis + description-update and then hands off to `jira-executor`.
- One child at a time, with full user confirmation between children.
- The epic transitions to **In Progress** when the first child starts work (not when the last one finishes).
- The epic-level wrap-up comment aggregates per-child JIRA comment URLs — it is a pointer index, not a retelling of each child's implementation log.

## Per-mode rule — Remember answers for the session {#exe.remember}

Every `AskUserQuestion` prompt raised by this mode MUST follow the Human Confirmation Protocol's **Remember Answer for the Session** rule (see `.claude/skills/jira/_shared/references/human-confirmation-protocol.md` — section [`hcp.8.remember-answer`]).

Concretely:

- The between-children "Continue / Pause" prompt (Stage 3) offers a remember variant so the user can lock in "always continue" for the rest of the epic execution.
- The Stage 1 queue confirmation is typically asked once and is not remembered, but any prompt that repeats across children must offer a remember variant.
- Destructive actions (Epic → Done transition, epic-level push if any) are **never** remembered — they are always re-asked per `hcp.6.destructive-pattern`.
- Downstream skills (`jira-executor`, `jira-story-executor`, `jira-bug-executor`) retain their own `hcp.8` handling for their own prompts.

## Contents

When starting each stage, display the section ID in the form `▶ [exe.N.name] — Stage title`. The `exe.` prefix is used throughout.

| ID | Stage |
|---|---|
| [`exe.1.load-epic-context`](#exe.1.load-epic-context) | Load Epic Context |
| [`exe.2.child-execution`](#exe.2.child-execution) | Child-by-Child Execution (delegated) |
| [`exe.3.next-child`](#exe.3.next-child) | Next Child |
| [`exe.4.epic-completion`](#exe.4.epic-completion) | Epic Completion |

## Principle — Single Execution Path {#exe.principle}

`execute.md` does NOT reimplement per-ticket work. It only:

1. Loads the epic and its children.
2. Filters to pending children.
3. For each pending child, announces the handoff and reads the appropriate skill file:
   - **Story** → `.claude/skills/jira/jira-story-executor/SKILL.md`
   - **Task / Subtask** → `.claude/skills/jira/jira-executor/SKILL.md`
   - **Bug** → `.claude/skills/jira/jira-bug-executor/SKILL.md`
4. Begins that skill's flow from Step 1, passing the child ticket key.
5. After the child returns, prompts the user to continue / stop / revisit.
6. When all pending children are done (or user stops), closes the epic with a final comment and transition.

All implementation, spec-first, dev flows, plans, code, In-Progress transitions, git commits, and per-ticket JIRA updates happen inside the delegated skill(s) — never inline here.

---

## Stage 1 — Load Epic Context {#exe.1.load-epic-context}

### Step 1.1 — Fetch Epic and Children

Use `getJiraIssue` to fetch the epic by the provided key. Fetch all child issues using `searchJiraIssuesUsingJql`:

```
JQL: "Epic Link" = <EPIC-KEY> ORDER BY rank ASC
```

(Adjust to the Jira instance's parent-link field if necessary — `parent = <EPIC-KEY>` also works on modern Jira cloud.)

Extract for each child:

- Key, issue type (Story / Task / Subtask / Bug), summary, status, priority

### Step 1.2 — Filter to Pending Children

Remove children with status = Done.

Present the overview:

```
▶ [exe.1.load-epic-context] — Load Epic Context

═══════════════════════════════════════════════════════
EPIC: [KEY] — [Title]
═══════════════════════════════════════════════════════

Type:        Epic
Status:      [Current Status]

Pending children: [pending] of [total]

 #  | Key       | Type    | Priority | Status  | Summary                 | Routed via
────┼───────────┼─────────┼──────────┼─────────┼─────────────────────────┼────────────────────────────
 1  | ACD-11    | Story   | High     | To Do   | Build signup flow       | jira-story-executor → jira-executor
 2  | ACD-12    | Task    | Medium   | To Do   | Add Postgres migration  | jira-executor
 3  | ACD-13    | Bug     | High     | Open    | Fix session expiry      | jira-bug-executor → jira-executor
 ...

═══════════════════════════════════════════════════════

Execution plan: each pending child is handed to the right downstream
skill one at a time. All per-ticket work (spec-first, dev flows,
implementation, commits, JIRA updates) happens inside those skills —
this orchestrator only iterates.
```

Use `AskUserQuestion`:

- **Yes, start executing** — Begin with the first pending child
- **Review first** — Show full description of each pending child, then re-ask
- **Re-order children** — Change the execution order
- **Cancel** — Stop the epic executor

If reordering, update the in-memory queue and re-present.

---

## Stage 2 — Child-by-Child Execution (delegated) {#exe.2.child-execution}

### Step 2.0 — Transition Epic to In Progress (once)

Before starting the **first** pending child of this session, transition the parent Epic to **In Progress** via `transitionJiraIssue` if it is not already In Progress or Done. This ensures the epic's own status reflects that real work is underway, without waiting for the final child to close. Skip this step on subsequent iterations of the Stage 2 loop.

**Destructive pattern applies** (status change on the epic) — always announce the transition; do not remember the decision across sessions.

For each pending child in the queue, in order:

### Step 2.1 — Announce the handoff

```
═══════════════════════════════════════════════════════
CHILD [current] of [total]: [KEY] — [Summary]
═══════════════════════════════════════════════════════

▶ [exe.2.child-execution] — Handing [KEY] to [skill]

Type:         [Story / Task / Subtask / Bug]
Routed via:   [jira-story-executor → jira-executor
               | jira-executor
               | jira-bug-executor → jira-executor]
```

### Step 2.2 — Delegate to the appropriate skill

Based on the child's issue type:

- **Story** — **Read `.claude/skills/jira/jira-story-executor/SKILL.md` and begin executing its flow from Step 1**, passing the story key. That skill loads the story's subtasks and hands each one to `jira-executor` in turn.

- **Task / Subtask** — **Read `.claude/skills/jira/jira-executor/SKILL.md` and begin executing its flow from Step 1**, passing the ticket key. `jira-executor` runs the full per-ticket flow (understand → dev-flows → spec-first → plan → implement → transition → commit → jira update).

- **Bug** — **Read `.claude/skills/jira/jira-bug-executor/SKILL.md` and begin executing its flow from Step 1**, passing the ticket key. The bug executor runs its analysis flow, gets user confirmation, writes the analysis back to the ticket description, and then hands off to `jira-executor` for the actual fix.

Do **not** repeat any understanding, spec-first, planning, coding, commit, or JIRA-comment logic inside this mode — that is exactly what the delegated skill exists for.

### Step 2.3 — After delegation returns

When the delegated skill returns, the child has been implemented, committed, transitioned, and commented on by `jira-executor` (directly or via `jira-story-executor` / `jira-bug-executor`). Capture the URL of the per-child JIRA comment — for Stories this is the story-level wrap-up comment produced by `jira-story-executor`; for Tasks/Subtasks/Bugs it is the per-ticket comment produced by `jira-executor` in its Step 9. Store these URLs in an in-memory map keyed by child ticket — Stage 4 aggregates them into the epic-level audit trail. Proceed to Stage 3.

---

## Stage 3 — Next Child {#exe.3.next-child}

Show epic progress:

```
▶ [exe.3.next-child] — Progress

Epic Progress:
  ✅ [1] ACD-11 — Build signup flow (Done via jira-story-executor → jira-executor)
  ✅ [2] ACD-12 — Add Postgres migration (Done via jira-executor)
  ▶️ [3] ACD-13 — Fix session expiry (next — routing via jira-bug-executor)
  ☐  [4] ACD-14 — Wire email-verify endpoint (pending)
```

Use `AskUserQuestion` (remember-variant allowed per `exe.remember`):

- **Continue to next child** — Hand the next pending child to the appropriate skill (per Stage 2)
- **Stop here for now** — Pause execution; go to Stage 4 for a partial wrap-up
- **Revisit a completed child** — Re-hand a prior child to its executor
- **Skip this child** — Mark current child as skipped and move on
- **Remember: always continue this session** — Don't ask between children for the rest of this session

If continuing, return to Stage 2 for the next child. If stopping, go to Stage 4.

---

## Stage 4 — Epic Completion {#exe.4.epic-completion}

### Step 4.1 — Check Completion

Fetch the epic again and check whether all child tickets are now Done.

- **All done** → Proceed to 4.2.
- **Some pending** → Proceed to 4.3 (partial completion).

### Step 4.2 — Full Epic Completion

**Destructive pattern applies** — transitioning the Epic to **Done** is always explicitly confirmed; the decision is never remembered across the session.

Use `AskUserQuestion`:

- **Yes, mark Done** — Transition the Epic to Done via `transitionJiraIssue`
- **Leave In Progress** — Keep the epic in In Progress (e.g., pending review)
- **Custom status** — Transition to a user-named status

Add a final comment to the epic using `addCommentToJiraIssue`, aggregating the per-child comment URLs captured in Step 2.3:

```
Epic implementation complete.

Summary:
- Children executed: [count]
  ✅ [KEY] — [summary] (via [skill])   (comment: [URL from Step 2.3])
  ✅ [KEY] — [summary] (via [skill])   (comment: [URL from Step 2.3])
  ...

Per-ticket implementation details (files changed, specs touched, dev
flows, commits) live on each child ticket's own JIRA comment — the
links above go directly to those authoritative logs. This epic-level
comment only aggregates pointers; it does not restate what the child
comments already say.

Epic ready for review.
```

Present the completion summary:

```
▶ [exe.4.epic-completion] — Epic Complete

═══════════════════════════════════════════════════════
EPIC COMPLETE
═══════════════════════════════════════════════════════

Epic:     [KEY] — [Title]
Status:   ✓ Done

Children: [N] executed via downstream skills
          (jira-executor / jira-story-executor / jira-bug-executor)

═══════════════════════════════════════════════════════
```

Use `AskUserQuestion`:

- **Done, thanks!** — End the session
- **Add final notes to the epic** — Append extra context
- **Move to another epic** — Provide the next epic key

### Step 4.3 — Partial Completion

If some children are still pending:

Keep the epic in its current status (should be In Progress from Step 2.0, unless manually overridden).

Add a progress comment to the epic, again aggregating the per-child comment URLs:

```
Execution paused.

Progress:
- Completed: [children listed, each with link to per-child comment from Step 2.3]
- Pending:   [children listed]

Resume with: execute epic [EPIC-KEY]
```

Present the summary:

```
▶ [exe.4.epic-completion] — Execution Paused

═══════════════════════════════════════════════════════
EXECUTION PAUSED
═══════════════════════════════════════════════════════

Epic:        [KEY] — [Title]
Status:      [current]

Completed:   [X] children (via downstream skills)
Pending:     [X] children

Resume anytime with: execute epic [KEY]

═══════════════════════════════════════════════════════
```

Use `AskUserQuestion`:

- **Done for now** — End the session
- **Actually, continue** — Resume execution (return to Stage 2 with the next pending child)
- **Add notes** — Additional context before ending

---

## Summary

**Mode 3 (Execute Epic)** is an orchestrator only:

1. **Load** the epic and its pending children.
2. **Iterate** the children — for each one, hand off to the right downstream skill:
   - Story → `jira-story-executor` → `jira-executor`
   - Task / Subtask → `jira-executor`
   - Bug → `jira-bug-executor` → `jira-executor`
3. **Prompt** between children (continue / stop / revisit / skip).
4. **Close** the epic with a summary comment and Done transition once all children are Done.

No implementation, spec-first, dev-flows, planning, coding, commits, or per-ticket JIRA comments happen inside this file — all of that is owned by `jira-executor` via the delegated skills.
