---
name: jira-epic-orchestrator
description: >
  MANDATORY TRIGGERS: "plan epic", "orchestrate epic", "execute epic",
  "update epic", "update and execute epic", "start jira-epic-orchestrator",
  "plan feature", "break down feature", "plan initiative", or any request
  to plan, create, execute, or modify a full feature hierarchy in JIRA.
  Use this skill whenever the user wants to go from a feature idea to JIRA
  tickets, implement an existing epic, or modify an epic's structure.
  This skill covers 5 modes: plan-only, plan+execute, execute-only,
  update-only, and update+execute. It scales hierarchy based on scope —
  small features → single Epic, large features → Initiative with multiple Epics.
  Even if the user just says "I have a feature idea" or "let's break this down",
  this skill is likely what they need.
---

<!-- ============================================================
     JIRA EPIC ORCHESTRATOR — ROUTER
     ============================================================
     This is the entry point. It detects the mode and dispatches
     to the correct mode file under ./modes/.
     
     Mode files:
       modes/plan.md       → Mode 1 (plan epic) — 8 stages
       modes/execute.md    → Mode 3 (execute epic) — 5 stages
       modes/update.md     → Mode 4 (update epic) — 4 stages
     
     Composite modes:
       Mode 2 (orchestrate epic)       → plan.md then execute.md
       Mode 5 (update and execute epic) → update.md then execute.md
     
     Hierarchy (auto-detected by scope):
       Small/medium → Epic → Stories → Tasks
       Large        → Initiative → Epics → Stories → Tasks
     
     Target JIRA project: ACD (Aqua Claude Dev)
     ============================================================ -->

# JIRA Epic Orchestrator

A full-lifecycle skill for planning, creating, executing, and modifying JIRA epics with deep technical specifications. It operates in 5 modes, each with its own trigger command. Each core mode lives in its own file under `modes/`.

**Project:** ACD (Aqua Claude Dev)

**Shared Protocols:** All mode files reference reusable protocols in `.claude/skills/jira/_shared/references/`. Read the relevant protocol file when a step says "follow [protocol-name]".

- `human-confirmation-protocol.md` — Enhanced confirmation patterns (used in ALL modes)
- `acceptance-criteria-checklist.md` — Suggest & select acceptance criteria (plan, update)
- `technical-deep-dive.md` — Interactive technical spec gathering (plan, update)
- `spec-first-protocol.md` — Verify .spec coverage before implementation (execute)
- `flow-protocol.md` — Detect and document user flows (execute)
- `implementation-protocol.md` — Standard implementation flow (execute)
- `git-workflow.md` — Git operations after implementation (execute, Level 2)

## Section IDs — Master Index

The orchestrator spans multiple mode files. Each mode uses its own prefix. This master index covers all IDs across all modes.

**Router prefix:** `orch`

```
▶ [orch.detect] — Mode Detection
```

**Mode 1 — Plan (prefix: `plan`):**

| ID | Stage |
|---|---|
| `plan.1.feature-discovery` | Feature Discovery |
| `plan.2.scope-assessment` | Scope Assessment & Hierarchy Proposal |
| `plan.3.epic-definition` | Epic Definition(s) |
| `plan.4.story-breakdown` | Story Breakdown |
| `plan.5.task-decomposition` | Task Decomposition & Technical Deep-Dive |
| `plan.6.full-plan-review` | Full Plan Review |
| `plan.7.output-generation` | Output Generation |
| `plan.8.batch-jira-creation` | Batch JIRA Creation |

**Mode 3 — Execute (prefix: `exe`):**

| ID | Stage |
|---|---|
| `exe.1.load-epic-context` | Load Epic Context |
| `exe.2.story-execution` | Story-by-Story Execution |
| `exe.3.git-commit` | Git Commit (Per Story) |
| `exe.4.next-story` | Next Story |
| `exe.5.epic-completion` | Epic Completion |

**Mode 4 — Update (prefix: `update`):**

| ID | Stage |
|---|---|
| `update.1.load-current-state` | Load Current State |
| `update.2.collect-modifications` | Collect Modifications |
| `update.3.review-modified-plan` | Review Modified Plan |
| `update.4.apply-changes` | Apply Changes to JIRA and Spec File |

---

## Commands & Mode Detection {#orch.detect}

Each mode has a specific trigger command:

| Command | Mode | Mode File | What It Does |
|---|---|---|---|
| `plan epic` | Mode 1 | `modes/plan.md` | Plan the feature hierarchy and create JIRA tickets. No code execution. |
| `orchestrate epic` | Mode 2 | `modes/plan.md` → `modes/execute.md` | Plan, create JIRA tickets, then immediately start executing. |
| `execute epic <EPIC-KEY>` | Mode 3 | `modes/execute.md` | Execute an existing epic — implement only pending/unfinished tickets. |
| `update epic <EPIC-KEY>` | Mode 4 | `modes/update.md` | Modify an existing epic's structure. No execution. |
| `update and execute epic <EPIC-KEY>` | Mode 5 | `modes/update.md` → `modes/execute.md` | Modify an existing epic, then execute pending tickets. |

If the user says `start jira-epic-orchestrator` without a specific command, ask using `AskUserQuestion`:

- **Plan epic** — Plan a new feature and create JIRA tickets (no execution)
- **Orchestrate epic** — Plan, create JIRA tickets, and start executing immediately
- **Execute epic** — Implement an existing epic (pending tickets only)
- **Update epic** — Modify an existing epic's structure (no execution)

---

## Mode Dispatch

### Mode 1 — Plan Epic

**Trigger:** `plan epic`

**Read and follow `.claude/skills/jira/jira-epic-orchestrator/modes/plan.md`.**

Runs all 8 planning stages (Feature Discovery → Scope Assessment → Epic Definition → Story Breakdown → Task Decomposition → Full Plan Review → Output Generation → Batch JIRA Creation). No code is written.

### Mode 2 — Orchestrate Epic

**Trigger:** `orchestrate epic`

This is a composite mode:

1. **Read and follow `.claude/skills/jira/jira-epic-orchestrator/modes/plan.md`** — Run all 8 planning stages.

2. **After JIRA tickets are created**, insert a confidence checkpoint (per Human Confirmation Protocol):

   ```
   Quick check before we move to execution:

   We've completed all planning and created [X] JIRA tickets.
   Next up: Start implementing the first pending story.

   Ready to begin execution?
   ```

   Use `AskUserQuestion`:
   - **Yes, start executing now** — Begin implementing the first epic/story
   - **No, stop here** — End the session (tickets are created, execute later)
   - **Yes, but I want to add context first** — Additional notes before execution begins

3. **If continuing, read and follow `.claude/skills/jira/jira-epic-orchestrator/modes/execute.md`** using the epic key(s) just created.

### Mode 3 — Execute Epic

**Trigger:** `execute epic <EPIC-KEY>` (e.g., `execute epic ACD-5`)

**Read and follow `.claude/skills/jira/jira-epic-orchestrator/modes/execute.md`.**

Implements pending stories/tasks one by one with spec-first checks, flow detection, and separate commits per story.

### Mode 4 — Update Epic

**Trigger:** `update epic <EPIC-KEY>` (e.g., `update epic ACD-5`)

**Read and follow `.claude/skills/jira/jira-epic-orchestrator/modes/update.md`.**

Modifies the epic's structure (add, remove, edit, restructure) and updates JIRA + spec files. No code is written.

### Mode 5 — Update and Execute Epic

**Trigger:** `update and execute epic <EPIC-KEY>` (e.g., `update and execute epic ACD-5`)

This is a composite mode:

1. **Read and follow `.claude/skills/jira/jira-epic-orchestrator/modes/update.md`** — Run all 4 update stages.

2. **After changes are applied**, insert a confidence checkpoint (per Human Confirmation Protocol):

   ```
   Quick check before we move to execution:

   We've updated the epic structure and applied all changes.
   Next up: Start implementing pending tickets.

   Ready to begin execution?
   ```

   Use `AskUserQuestion`:
   - **Yes, start executing now** — Begin implementing pending tickets
   - **No, stop here** — End the session (changes are saved)
   - **Yes, but I want to add context first** — Additional notes before execution begins

3. **If continuing, read and follow `.claude/skills/jira/jira-epic-orchestrator/modes/execute.md`** with the updated epic.

---

## Hierarchy Scaling

Auto-detected during planning (Mode 1/2) and updating (Mode 4):

**Signals that it needs an Initiative (multiple epics):**
- Feature spans 3+ distinct functional areas
- Capabilities are loosely coupled and could ship independently
- 6+ major capabilities listed

**Signals that a single Epic is enough:**
- Feature focused on one functional area
- Capabilities are tightly coupled
- 1-4 major capabilities listed

The user can override the recommendation in either direction.

---

## Important Principles

- **Never originate substance.** All decisions about what to build, business logic, and architecture come from the user. You propose, the user decides.
- **Scale the hierarchy to fit the scope.** Large features get Initiatives with multiple epics. Focused features get a single epic.
- **One question at a time.** Never combine multiple open-ended questions.
- **Always confirm before acting.** No JIRA tickets are created, no code is written, and no transitions happen without explicit user approval. Always include an "add more details" option per the Human Confirmation Protocol.
- **Technical specs are first-class.** Database models, migration scripts, API contracts, and component designs are part of the plan — not afterthoughts.
- **Spec-first execution.** In execution modes (3, 5), always run the Spec-First Protocol before implementing.
- **Acceptance criteria are suggested, not imposed.** Use the Acceptance Criteria Checklist Protocol to propose tailored criteria and let the user select, modify, or replace them.
- **Epics are independently deliverable.** Each epic should be a coherent unit of work that could ship on its own.
- **Only touch pending work.** In execution modes (3, 5), skip any tickets already marked Done.
- **How this relates to other skills:** `ticket-creator` creates single tickets. `executor` implements single tickets. `batch-executor` handles multiple standalone tickets. This orchestrator does full epic lifecycle — same discipline, bigger scope.
