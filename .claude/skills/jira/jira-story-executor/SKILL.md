---
name: jira-story-executor
description: >
  Use this skill whenever the user wants to execute a single JIRA Story and all
  of its subtasks in sequence. Trigger on "start jira-story-executor",
  "execute story", "implement story", "work on story", or commands like
  "execute story ACD-8". This skill is a **thin orchestrator**: it loads the
  story + all subtasks, presents the queue, and then delegates the actual
  per-subtask execution to `jira-executor` (one subtask at a time). All
  understanding, dev flows, spec-first checks, implementation, transitions,
  commits, and JIRA updates happen inside `jira-executor` — this skill never
  duplicates that logic.
---

# JIRA Story Executor

Execute a single JIRA Story and all its subtasks in sequence. This skill is responsible **only** for loading story context, presenting the subtask queue, transitioning the parent story at the right moments, and iterating. Each subtask is handed off to `jira-executor` (or `jira-bug-executor` for Bug subtasks) for the full understand → dev-flows → spec-first → plan → implement → transition → commit → JIRA-update cycle.

**Shared Protocols:** See `.claude/skills/jira/_shared/references/` for reusable protocol files referenced by `jira-executor`.

## Per-skill rule — Remember answers for the session {#story.remember}

Every `AskUserQuestion` prompt raised by this skill MUST follow the Human Confirmation Protocol's **Remember Answer for the Session** rule (see `.claude/skills/jira/_shared/references/human-confirmation-protocol.md` — section [`hcp.8.remember-answer`]).

Concretely:

- Non-destructive queue-navigation prompts (e.g., "Continue to next subtask?") offer a remember variant so the user can accept the flow once and cruise through the queue.
- When a remembered choice is in effect, print `▸ Using remembered choice for <prompt topic>: <value>` and skip the question.
- Destructive prompts (see `hcp.6.destructive-pattern`) are NEVER remembered — Story → Done transition, story-level push, and any destructive wrap-up action is always re-asked.

## Section IDs

**Prefix:** `story`

When starting each stage, display the section ID:

```
▶ [story.N.name] — Stage title
```

| ID | Stage |
|---|---|
| `story.1.load-context` | Load Story Context |
| `story.2.subtask-execution` | Per-Subtask Execution (delegated to `jira-executor`) |
| `story.3.story-completion` | Story Completion |

## Principle — Single Execution Path {#story.principle}

`jira-story-executor` does NOT reimplement the execution flow. It only:

1. Loads the story and its subtasks.
2. Filters to pending subtasks.
3. For each pending subtask, **re-detects the issue type** and hands control to the correct executor (`jira-bug-executor` for Bug subtasks, `jira-executor` for everything else) by reading the right `SKILL.md` and starting from Step 1.
4. Transitions the parent Story to In Progress when the first subtask begins real work.
5. After all subtasks are complete (or the user stops), wraps up the story status + final comment, aggregating the per-subtask comment URLs for a single audit trail.

This means the spec-first check, dev flows, implementation plan, code changes, In-Progress transition on the subtask, git commits, and the per-subtask JIRA comment all live inside `jira-executor` — exactly as they do for a standalone Task. No duplication.

## Resume semantics {#story.resume}

`execute story <STORY-KEY>` may be run repeatedly against the same story. On resume:

- Already-Done subtasks are listed with ✅ and skipped — their executors ran on a previous invocation.
- In-Progress subtasks resume where they left off: `jira-executor` will detect that `.spec/dev-flows/feature/<feature-name>/code-to-phrase.md` already exists and ask whether to reuse or regenerate (see `jira-executor` Step 2 for skip-if-exists handling). This skill does not re-prompt for those artefacts itself.
- Open subtasks are queued as usual.
- The parent Story's current status is read from JIRA — if it's already In Progress, Step 2.0 below does not transition it again.

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
- All child subtasks (key, summary, **issue type**, status, priority)
- Story current status

Record each subtask's **issue type** explicitly — this drives the per-subtask routing decision in Step 2.1.

### Step 1.2 — Filter to Pending Subtasks

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

Subtasks: [total] total, [pending] pending, [done] already done

 #  | Key       | Type    | Priority | Status  | Summary
────┼───────────┼─────────┼──────────┼─────────┼──────────────────────────
 1  | ACD-18    | Subtask | Medium   | Open    | Initialize Next.js project
 2  | ACD-19    | Bug     | High     | Open    | Fix login redirect loop
 ...

═══════════════════════════════════════════════════════

Execution plan:
  • Story transitions to "In Progress" when the first pending subtask starts.
  • Each pending subtask is routed by issue type:
      - Bug      → `jira-bug-executor` (analysis) → `jira-executor` (fix)
      - Task     → `jira-executor`
      - Subtask  → `jira-executor`
  • The chosen executor runs the full flow per subtask (understand →
    dev-flows → spec-first → plan → implement → transition → commit →
    jira update).
```

If any subtasks are already Done, mark them with ✅ in the list.

Use `AskUserQuestion` (remember-variant allowed per `story.remember`):

- **Yes, start executing** — Begin handing the first pending subtask to the right executor
- **Review first** — Show full description of each pending subtask, then re-ask
- **Re-order subtasks** — Let me change the order before starting
- **Cancel** — Stop the story executor

If the user reorders, update the in-memory queue and re-present.

---

## Stage 2 — Per-Subtask Execution (delegated to `jira-executor`) {#story.2.subtask-execution}

### Step 2.0 — Transition Story to In Progress (once)

Before starting the **first** pending subtask of this session, transition the parent Story to **In Progress** via `transitionJiraIssue` if it is not already In Progress or Done. This makes the Story's own status reflect that real work has begun, without waiting for the last subtask. Skip this step on subsequent iterations of the Step 2 loop.

**Destructive pattern applies** (status change on the parent story) — always announce the transition to the user; do not remember the decision across sessions.

For each pending subtask in the queue:

### Step 2.1 — Re-detect issue type and announce the handoff

For each subtask, re-read its issue type from the data captured in Step 1.1. **Do not assume all children are Subtask type** — projects frequently attach Bugs and Tasks as children of a Story.

Routing table:

| Subtask issue type | First executor              | Final executor    |
|--------------------|-----------------------------|-------------------|
| Bug                | `jira-bug-executor`         | `jira-executor`   |
| Task / Subtask     | `jira-executor`             | `jira-executor`   |
| Anything else      | Ask the user before routing | —                 |

Announce:

```
═══════════════════════════════════════════════════════
SUBTASK [current] of [total]: [KEY] — [Summary]   (type: [Bug|Task|Subtask])
═══════════════════════════════════════════════════════

▶ [story.2.subtask-execution] — Routing subtask [KEY] to `[chosen executor]`

`[chosen executor]` will now run its full flow for this subtask:
  • Step 1 — Fetch and present ticket understanding
  • Step 2 — Generate dev flows (code-to-phrase + code-to-sentence)
  • Step 3 — Spec-first check
  • Step 4 — Identify specs to update
  • Step 5 — Present implementation plan (every item tagged to a phrase)
  • Step 6 — Implement the code (strict 1:1 with the phrase list)
  • Step 7 — Transition to In Progress (after code ready)
  • Step 8 — Git workflow (separate commits for spec / dev-flows / code)
  • Step 9 — Post JIRA update
```

### Step 2.2 — Delegate to the correct executor

For **Bug** subtasks: read `.claude/skills/jira/jira-bug-executor/SKILL.md` and begin executing from its Step 1, passing the subtask key. That skill will, on its own, hand off to `jira-executor` once the analysis block is confirmed.

For **all other** subtasks: read `.claude/skills/jira/jira-executor/SKILL.md` and begin executing its flow from Step 1, passing the subtask key.

All understanding, dev flows, spec checks, planning, implementation, transitions, and JIRA updates happen inside those skills. Do **not** repeat any of those steps in this skill — that is exactly what those skills exist for.

### Step 2.3 — After the executor finishes this subtask

When the executor returns, the subtask has been implemented, committed, transitioned, and commented on. Capture the URL of the per-subtask JIRA comment (if returned by `addCommentToJiraIssue` / readable via `getJiraIssue`) and store it in an in-memory list keyed by subtask — Stage 3 will aggregate these into the story-level audit trail.

Show progress:

```
Subtask Progress:
  ✅ [1] ACD-18 — Initialize Next.js project (Done)
  ✅ [2] ACD-19 — Fix login redirect loop (Done — routed via jira-bug-executor)
  ▶️ [3] ACD-20 — Configure ESLint & Prettier (next)
  ☐  [4] ACD-21 — Install core dependencies (pending)
  ☐  [5] ACD-22 — Set up base folder structure (pending)
```

Use `AskUserQuestion` (remember-variant allowed — "Always continue automatically this session" is a valid remember option):

- **Continue to next subtask** — Hand the next pending subtask to the right executor
- **Stop here for now** — Pause execution; go to Stage 3 for partial-completion wrap-up
- **Revisit a completed subtask** — Reopen a prior subtask (re-hand to the executor)
- **Remember: always continue this session** — Don't ask between subtasks for the rest of this session

If continuing, return to Step 2.1 for the next subtask. If stopping, proceed to Stage 3.

---

## Stage 3 — Story Completion {#story.3.story-completion}

### Step 3.1 — Check completion status

Check if ALL subtasks are marked Done.

- **All done** → Proceed to 3.2 (full completion)
- **Some pending** → Proceed to 3.3 (partial completion)

### Step 3.2 — Full Story Completion

**Destructive pattern applies** — transitioning the Story to **Done** is always explicitly confirmed; this decision is never remembered across the session.

Use `AskUserQuestion`:

- **Yes, mark Done** — Transition the Story to Done via `transitionJiraIssue`
- **Leave In Progress** — Keep the story in In Progress (e.g., awaiting review)
- **Custom status** — Transition to a status the user names

Add a final comment to the story using `addCommentToJiraIssue`, aggregating the per-subtask comment URLs captured in Step 2.3:

```
All subtasks completed.

Summary:
- Subtasks implemented: [count]
- Subtask results:
  ✅ [KEY] — [summary]   (comment: [URL from Step 2.3])
  ✅ [KEY] — [summary]   (comment: [URL from Step 2.3])
  ...

Each subtask was executed by its dedicated executor, which recorded its
own per-ticket update (files changed, specs touched, dev flows, commits).
The links above go directly to those per-subtask comments — they are the
authoritative implementation log. This story-level comment only aggregates.

Story ready for review.
```

Present the completion summary:

```
▶ [story.3.story-completion] — Story Complete

═══════════════════════════════════════════════════════
STORY COMPLETE
═══════════════════════════════════════════════════════

Story:          [KEY] — [Title]
Status:         [Done | In Progress | Custom]
Parent Epic:    [EPIC-KEY] — [Title]

Subtasks:       [N] implemented via their dedicated executors

═══════════════════════════════════════════════════════
```

Use `AskUserQuestion`:

- **Done, thanks!** — End the session
- **Add final notes to the story** — Append extra context to the story comment
- **Execute the next story** — Provide the next story key to pick up

### Step 3.3 — Partial Completion

If some subtasks are still pending (user stopped early):

Keep the story in its current status (should be In Progress from Step 2.0, unless the user chose a different status manually).

Add a progress comment to the story, again aggregating the per-subtask comment URLs:

```
Execution paused.

Progress:
- Completed: [subtasks listed, with links to their per-subtask comments]
- Pending:   [subtasks listed]

Resume with: execute story [STORY-KEY]
```

Present the progress summary:

```
▶ [story.3.story-completion] — Execution Paused

═══════════════════════════════════════════════════════
EXECUTION PAUSED
═══════════════════════════════════════════════════════

Story:       [KEY] — [Title]
Status:      [current]

Completed:   [X] subtasks (via their dedicated executors)
Pending:     [X] subtasks

Resume anytime with: execute story [KEY]

═══════════════════════════════════════════════════════
```

Use `AskUserQuestion`:

- **Done for now** — End the session
- **Actually, continue** — Resume (return to Stage 2 with the next pending subtask)
- **Add notes** — Additional context before ending

---

## Important Principles

- **All per-subtask work runs through `jira-executor`.** Bug subtasks detour briefly through `jira-bug-executor` for root-cause analysis, then return to `jira-executor`. This skill never implements code, never runs spec-first, never generates dev flows, never commits, never posts per-subtask JIRA comments. It only orchestrates the queue.
- **Re-detect issue type per subtask.** Do not assume everything under a story is type "Subtask". Inspect the `issuetype` field for each child and route accordingly.
- **Story → In Progress at first subtask start.** Don't wait for completion to reflect reality on the parent ticket.
- **One subtask at a time.** Finish one subtask entirely (via its executor's full flow) before starting the next.
- **Story-level wrap-up only.** This skill owns the story-level final comment (which aggregates per-subtask comment URLs) and the story-level Done transition — not the per-subtask ones.
- **Never originate substance.** All substance comes from the subtasks, specs, and the downstream executor's own confirmation points.
- **Always confirm before acting.** No subtask handoff without explicit user approval; destructive actions (Story Done transition) never carry a remember-for-session option.
- **Subtask order matters.** Process subtasks in creation order (ascending) unless the user reorders during Stage 1.
