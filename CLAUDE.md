# JIRA-Driven Claude Code Approach for SaaS (Vibe Engineering) {#claude}

We follow a development approach that combines three pillars to build and maintain our SaaS platform. Each pillar has a clearly defined role and boundary.

## 1. JIRA — What to develop {#claude.1.jira}

JIRA is the preferred source of truth for **what** the team is building. **Ask for a JIRA ticket first**, but allow the user to proceed without one when they explicitly choose to.

- **Default path:** every feature, bug fix, or task originates as a JIRA ticket. The ticket captures the requirements, acceptance criteria, and priority.
- **First question on any change request** is always "Is there a JIRA ticket for this, or should we create one?" — never silently skip the ask.
- **No-ticket path:** if the user explicitly says they want to continue without a JIRA ticket, honour that. `jira-executor` has a no-ticket branch (see its Step 9 branching): the work still produces the `code-to-phrase` / `code-to-sentence` log so the change remains auditable, it just isn't posted back to JIRA.
- Even in no-ticket mode, all the other pillars still apply — SPECs must cover the change, and Claude Cowork is still the only thing writing the code.

## 2. SPECs — How to develop {#claude.2.spec-config}

The SPECs library (hierarchical `CLAUDE.md` indexes plus `.spec/` files) is the single source of truth for **how the system works**.

- Covers contracts, schemas, rules, architecture decisions, and naming/style conventions.
- Organised as hierarchical `CLAUDE.md` files that serve as indexes at each level of the project; the authoritative content lives under `.spec/dev/`, and per-ticket implementation logs live under `.spec/dev-flows/`.
- This root `CLAUDE.md` is the top-level entry point; module-level `CLAUDE.md` files provide deeper context as the project grows.
- No code is written against a SPEC gap. If something isn't in the SPEC, `jira-executor`'s Step 3 spec-first gate will surface it and require it to be added (with human confirmation) before implementation can start.

## 3. Claude Cowork — Actual develop {#claude.3.claude-code}

Claude Cowork is the disciplined implementation partner that does the **actual development work** — it writes, edits, and deletes code; produces the dev-flow artefacts; runs the spec-first audit; runs git; and posts JIRA updates. It operates strictly within the contract that JIRA and SPECs define.

- **Writes code** based on requirements defined in JIRA tickets (or the agreed no-ticket description) and specifications in the SPECs library.
- **Produces the per-ticket dev-flow** (`code-to-phrase` + `code-to-sentence`) before touching any code, and implements strictly 1:1 against the confirmed phrase list. No extra files, tests, refactors, imports, configs, or "while we're here" tidy-ups.
- **Formats and organises human-provided content** into JIRA tickets and SPEC files.
- **Never originates substance.** All decisions about _what_ to build, _why_ to build it, and _how_ the system should behave come from humans via JIRA and the SPECs library. Claude Cowork executes — it does not decide.

## Boundaries {#claude.4.boundaries}

The three pillars own three distinct responsibilities. The table below spells out who owns each step so the boundary between human intent and Claude Cowork's execution is never ambiguous.

| # | Responsibility                                                                                        | Pillar          | Owner                                                                 |
| - | ----------------------------------------------------------------------------------------------------- | --------------- | --------------------------------------------------------------------- |
| 1 | Deciding **what** to build (features, bug fixes, priorities, acceptance criteria)                     | JIRA            | **Humans** — via a JIRA ticket, or via an explicit no-ticket ask      |
| 2 | Deciding **how** the system works (contracts, schemas, architecture, naming, business rules)          | SPECs           | **Humans** — decisions live in `.spec/dev/` and in `CLAUDE.md` indexes |
| 3 | Deciding whether a given change may skip the JIRA ticket (the no-ticket opt-out)                       | JIRA            | **Humans only** — default is always ticket-first; never auto-skipped  |
| 4 | Drafting the per-ticket `code-to-phrase` / `code-to-sentence` dev-flow from the ticket + SPECs        | Claude Cowork   | **Claude Cowork** — drafts; a human must confirm before coding        |
| 5 | **Confirming** the dev-flow (it becomes the authoritative scope of the change)                         | JIRA / SPECs    | **Humans** — confirmation locks the phrase list as the contract       |
| 6 | Running the spec-first audit and surfacing missing / incomplete specs                                  | SPECs           | **Claude Cowork** — surfaces; humans author the actual spec content   |
| 7 | Writing, editing, and deleting code **strictly 1:1 with the confirmed phrase list** (no extras)         | Claude Cowork   | **Claude Cowork**                                                     |
| 8 | Git (commits, push, branches) and JIRA updates (status transitions, comments) for the change            | Claude Cowork   | **Claude Cowork** — with human confirmation on destructive actions    |
| 9 | Originating new requirements, business logic, or architecture decisions not already in JIRA or SPECs   | —               | **Humans only** — Claude Cowork never originates substance            |

**Rule of thumb:** if a decision is about *what*, *why*, or *whether to skip*, it is a human decision. If a decision is about *how to faithfully execute* an already-confirmed human decision, it is Claude Cowork's.

## Mandatory Ticket-First Rule {#claude.5.ticket-first}

> **CRITICAL: Ask for a JIRA ticket first. Only skip if the user explicitly opts out.**

Every change that touches the project folder should have a corresponding JIRA ticket **before** any implementation starts, unless the user explicitly chooses the no-ticket path. This includes:

- New features, bug fixes, and enhancements
- Spelling corrections, typo fixes, or formatting changes in project source
- `.spec` file creation or updates
- Installing, upgrading, or removing any library or dependency
- Any file creation, modification, or deletion in project source (code, assets, specs, flows, etc.)

**Exceptions (no ticket required):**

- **Anything under `.claude/`** — all files and folders under `.claude/` (create, edit, delete). This includes skill definitions under `.claude/skills/`, rule/config files such as `.claude/jira/**`, and any other meta-configuration. `.claude/` governs how Claude Cowork operates and is not product content.
- **This root `CLAUDE.md`** — the file you are reading. It is also meta-configuration and can be edited directly.
- **Explicit no-ticket ask** — the user says "continue without a JIRA ticket" (or similar). The change still goes through `jira-executor`'s flow (dev-flows, spec-first, strict scope, git) but the JIRA-specific steps are skipped. Acknowledge the choice before proceeding.

Anything outside these exceptions still requires a ticket first. When in doubt, treat the change as requiring a ticket and ask.

## Required Workflow (Strictly Enforced) {#claude.6.required-workflow}

The ticket-first rule applies to **every** issue type — Epic, Story, Task, Bug, or Subtask. What type the work is recorded as does not affect whether a ticket is required; the only question is **which entry point to use**, and that is decided by the *scope* of the ask, not the type label.

### Entry point — pick based on scope {#claude.6.1.entry-point}

Before creating anything, assess the scope of the request:

| Scope                                                                                                | Entry point                                       | Why                                                                                           |
| ---------------------------------------------------------------------------------------------------- | ------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| **Small** — the ask is clearly one unit of work that fits inside a single ticket                     | `start jira-ticket-creator`                       | Produces one Task / Bug / Story / Subtask / Epic-shell ticket.                                |
| **Big** — the ask feels larger than one ticket, or naturally breaks into multiple pieces of work     | `plan epic` (routes to `jira-epic-orchestrator`)  | Produces an Epic + child Stories/Tasks with parent/child linkage and a clear execution order. |
| **No-ticket (explicit)** — user declines the JIRA ticket and asks Claude Cowork to proceed directly  | `start jira-executor` (with no ticket key)        | Runs the full dev-flow + spec-first + strict-scope + git discipline without JIRA round-trips. |

**If unsure, ask the user before deciding.** When the ask feels big — or you notice you'd struggle to fit everything under a single ticket's acceptance criteria — default to offering `plan epic`. A lone Epic shell is rarely the right answer; if the natural type for the ask is "Epic", that's itself a signal to use `plan epic` instead of `jira-ticket-creator`.

### Sequence {#claude.6.2.sequence}

Every request that involves a change to the project must follow this sequence:

1. **Ask about JIRA first** — "Is there a ticket for this, or should we create one?" Only if the user explicitly declines does the no-ticket branch apply.
2. **Pick the entry point** — Small ask → `start jira-ticket-creator`. Big ask → `plan epic`. No-ticket ask → go straight to `start jira-executor`. When uncertain, ask the user. No code, no file edits, no installs until either a ticket exists or the user has confirmed the no-ticket path.
3. **Create the ticket(s)** — Run the chosen entry-point skill to create the ticket (or Epic + children) in JIRA. Skip in no-ticket mode.
4. **Ask for human confirmation** — Present what was created and explicitly ask: _"Should I proceed with execution?"_ Wait for approval before continuing.
5. **Pick the right executor** — Route through the **jira-router** or use the appropriate direct trigger to select the correct executor skill for the ticket type. All per-ticket execution ultimately funnels into `jira-executor`.
6. **Verify `.spec` files** — `jira-executor`'s spec-first gate checks whether any `.spec` files need to be created or updated for the change. If yes, update `.spec` first.
7. **Implement the change** — Only now execute the actual code/file changes, strictly against the confirmed `code-to-phrase` list.

**Human confirmation is required at every decision point.** When in doubt, ask — never assume.

## JIRA Skills {#claude.7.jira-skills}

All JIRA skills live under `.claude/skills/jira/`. When a trigger command is used, read the corresponding `SKILL.md` and **immediately begin the flow defined inside it**.

### Smart Router (Primary Entry Point) {#claude.7.1.smart-router}

**When the user mentions any JIRA ticket ID** (e.g., `implement ACD-18`, `execute ACD-5`, `ACD-18 give steps`, or just `ACD-18`), use the **jira-router** skill. It fetches the ticket, detects the issue type, and auto-routes to the correct upstream skill. **All per-ticket implementation ultimately runs through `jira-executor`** — upstream skills orchestrate, context-load, or analyse, and then hand off the actual work to `jira-executor`:

| Issue Type          | Upstream skill                                                          | Ultimately executes via       | Equivalent Command                |
| ------------------- | ----------------------------------------------------------------------- | ----------------------------- | --------------------------------- |
| **Epic**            | `jira-epic-orchestrator` (execute mode — iterates children)             | `jira-executor` (per child)   | `execute epic <KEY>`              |
| **Story**           | `jira-story-executor` (iterates subtasks)                               | `jira-executor` (per subtask) | `execute story <KEY>`             |
| **Task**            | `jira-executor` (direct)                                                | `jira-executor`               | `start jira-executor <KEY>`       |
| **Subtask**         | `jira-executor` (direct)                                                | `jira-executor`               | `start jira-executor <KEY>`       |
| **Bug**             | `jira-bug-executor` (analysis → confirm → update description → handoff) | `jira-executor`               | `start jira-bug-executor <KEY>`   |

Router skill: [`.claude/skills/jira/jira-router/SKILL.md`](.claude/skills/jira/jira-router/SKILL.md)

### Direct Triggers {#claude.7.2.direct-triggers}

These explicit commands still work and bypass the router:

| Trigger                                         | Skill File                                                                                                   | Description                                     |
| ----------------------------------------------- | ------------------------------------------------------------------------------------------------------------ | ----------------------------------------------- |
| `start jira-ticket-creator`                     | [`.claude/skills/jira/jira-ticket-creator/SKILL.md`](.claude/skills/jira/jira-ticket-creator/SKILL.md)       | Interactive JIRA ticket creation                |
| `start jira-executor <ticket-id>`               | [`.claude/skills/jira/jira-executor/SKILL.md`](.claude/skills/jira/jira-executor/SKILL.md)                   | Task/Subtask implementation (or no-ticket mode if called without a key) |
| `start jira-bug-executor <ticket-id>`           | [`.claude/skills/jira/jira-bug-executor/SKILL.md`](.claude/skills/jira/jira-bug-executor/SKILL.md)           | Bug investigation and fix                       |
| `execute story <STORY-KEY>`                     | [`.claude/skills/jira/jira-story-executor/SKILL.md`](.claude/skills/jira/jira-story-executor/SKILL.md)       | Execute a story and all its subtasks            |
| `start jira-story-executor <STORY-KEY>`         | [`.claude/skills/jira/jira-story-executor/SKILL.md`](.claude/skills/jira/jira-story-executor/SKILL.md)       | Same as `execute story` — alternate trigger     |
| `start jira-batch-executor <KEY-1, KEY-2, ...>` | [`.claude/skills/jira/jira-batch-executor/SKILL.md`](.claude/skills/jira/jira-batch-executor/SKILL.md)       | Execute multiple standalone tickets in sequence |
| `plan epic`                                     | [`.claude/skills/jira/jira-epic-orchestrator/SKILL.md`](.claude/skills/jira/jira-epic-orchestrator/SKILL.md) | Plan and create JIRA tickets — no code          |
| `orchestrate epic`                              | [`.claude/skills/jira/jira-epic-orchestrator/SKILL.md`](.claude/skills/jira/jira-epic-orchestrator/SKILL.md) | Plan, create tickets, then execute              |
| `execute epic <EPIC-KEY>`                       | [`.claude/skills/jira/jira-epic-orchestrator/SKILL.md`](.claude/skills/jira/jira-epic-orchestrator/SKILL.md) | Execute pending stories/tasks in an epic        |
| `update epic <EPIC-KEY>`                        | [`.claude/skills/jira/jira-epic-orchestrator/SKILL.md`](.claude/skills/jira/jira-epic-orchestrator/SKILL.md) | Modify epic structure — no execution            |
| `update and execute epic <EPIC-KEY>`            | [`.claude/skills/jira/jira-epic-orchestrator/SKILL.md`](.claude/skills/jira/jira-epic-orchestrator/SKILL.md) | Modify epic, then execute pending tickets       |
| `start flow-creator`                            | [`.claude/skills/jira/jira-flow-creator/SKILL.md`](.claude/skills/jira/jira-flow-creator/SKILL.md)           | Standalone flow documentation                   |

## Registered Specs {#claude.8.registered-specs}

Specs live under `.spec/dev/` (rules, architecture, epics, features — the long-lived contract of **how the system works**) and are referenced here so any skill can locate them. Per-feature dev-flow artefacts live under `.spec/dev-flows/feature/<feature-name>/` and are registered in section `claude.10.dev-flows` below, not here.

| Spec ID                         | File                                                                           | Description                                                            |
| ------------------------------- | ------------------------------------------------------------------------------ | ---------------------------------------------------------------------- |
| `naming-rule`                   | [`.spec/dev/rules/naming-rules.md`](.spec/dev/rules/naming-rules.md)           | Naming conventions — folders, files, components, variables            |

Add a new row here whenever a new spec is registered under `.spec/dev/` (epics, features, architecture, business rules, etc.).

## Registered Flows {#claude.9.registered-flows}

**"Flow"** in this project has two distinct meanings. Keep them separate — they live in different folders and are produced by different skills.

| Flow type | Folder | Produced by | Purpose |
| --------- | ------ | ----------- | ------- |
| **User flow** (user-facing journey + tests) | `.flows/<module>/<flow-name>/` — `flow.md`, `e2e.yaml`, `unit.yaml` | `jira-flow-creator` | Describes an end-user journey (sign-up, checkout, forgot-password UX) and its test scenarios. Long-lived, one per user-visible feature. |
| **Dev flow** (per-ticket implementation log) | `.spec/dev-flows/feature/<feature-name>/` — `code-to-phrase.md`, `code-to-sentence.md` (when stored as spec files; otherwise as JIRA comments on the ticket) | `jira-executor` (Step 2) | The ordered list of code changes for a specific ticket, in execution order. Serves as the authoritative checklist the executor follows. Stored either as spec files or as JIRA comments per the user's session-remembered choice — see `.claude/jira/jira-executor-rules.md`. |

No user flows registered yet.

## Registered Dev Flows {#claude.10.dev-flows}

No dev flows registered yet. Each row below points to a per-feature `code-to-phrase` / `code-to-sentence` artefact (or the JIRA ticket that carries them as comments).

| Feature name | Artefact location | Linked JIRA ticket |
| ------------ | ----------------- | ------------------ |
