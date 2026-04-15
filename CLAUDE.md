# JIRA-Driven Claude Code Approach for SaaS (Vibe Engineering)

> **JIRA Ticket:** [MD-11](https://vinay251101.atlassian.net/browse/MD-11)

## Overview {#cmd.1.overview}

We follow a development approach that combines three pillars to build and maintain our SaaS platform. Each pillar has a clearly defined role and boundary.

## The Three Pillars {#cmd.2.three-pillars}

### 1. JIRA — What {#cmd.2.1.jira}

JIRA is the single source of truth for **what** the team is building.

- Every feature, bug fix, and task originates as a JIRA ticket.
- Tickets capture the requirements, acceptance criteria, and priority.
- No work begins without a corresponding JIRA ticket.

### 2. Spec Config Markdown Library — How {#cmd.2.2.spec-config}

The Spec Config Markdown Library (hierarchical `CLAUDE.md` indexes) is the single source of truth for **how the system works**.

- Covers contracts, schemas, rules, and architecture decisions.
- Organized as hierarchical `CLAUDE.md` files that serve as indexes at each level of the project.
- This root `CLAUDE.md` is the top-level entry point; module-level `CLAUDE.md` files provide deeper context as the project grows.

### 3. Claude Code — Disciplined Implementation Partner {#cmd.2.3.claude-code}

Claude Code is a disciplined implementation partner that:

- **Writes code** based on requirements defined in JIRA tickets and specifications in the Markdown library.
- **Formats and organizes human-provided content** into JIRA tickets and MD files.
- **Never originates substance.** All decisions about _what_ to build, _why_ to build it, and _how_ the system should behave come from humans via JIRA and the Spec Config Markdown Library. Claude Code executes — it does not decide.

## Boundaries {#cmd.3.boundaries}

| Responsibility                                                          | Owner                                                    |
| ----------------------------------------------------------------------- | -------------------------------------------------------- |
| Deciding what to build                                                  | Humans via **JIRA**                                      |
| Defining how the system works (contracts, schemas, rules, architecture) | Humans via **Spec Config Markdown Library**              |
| Writing code that implements the above                                  | **Claude Code**                                          |
| Formatting/organizing content into tickets and MD files                 | **Claude Code**                                          |
| Originating new requirements, business logic, or architecture decisions | **Humans only** — Claude Code never originates substance |

## Mandatory Ticket-First Rule {#cmd.4.ticket-first}

> **CRITICAL: No work begins without a JIRA ticket. No exceptions.**

Every change that touches the project folder must have a corresponding JIRA ticket **before** any implementation starts. This includes:

- New features, bug fixes, and enhancements
- Spelling corrections, typo fixes, or formatting changes in project source
- `.spec` file creation or updates
- Installing, upgrading, or removing any library or dependency
- Any file creation, modification, or deletion in project source (code, assets, specs, flows, etc.)

**Exceptions (no ticket required):**

- **Skill definitions** — any file under `.claude/skills/` (create, edit, delete). Skills are meta-configuration that governs how Claude operates, not product content.
- **This root `CLAUDE.md`** — the file you are reading. It is also meta-configuration and can be edited directly.

Anything outside these two exceptions still requires a ticket first. When in doubt, treat the change as requiring a ticket and ask.

## Required Workflow (Strictly Enforced) {#cmd.5.required-workflow}

The ticket-first rule applies to **every** issue type — Epic, Story, Task, Bug, or Subtask. What type the work is recorded as does not affect whether a ticket is required; the only question is **which entry point to use**, and that is decided by the *scope* of the ask, not the type label.

### Entry point — pick based on scope {#cmd.5.1.entry-point}

Before creating anything, assess the scope of the request:

| Scope                                                                                                | Entry point                                       | Why                                                                                           |
| ---------------------------------------------------------------------------------------------------- | ------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| **Small** — the ask is clearly one unit of work that fits inside a single ticket                     | `start jira-ticket-creator`                       | Produces one Task / Bug / Story / Subtask / Epic-shell ticket.                                |
| **Big** — the ask feels larger than one ticket, or naturally breaks into multiple pieces of work     | `plan epic` (routes to `jira-epic-orchestrator`)  | Produces an Epic + child Stories/Tasks with parent/child linkage and a clear execution order. |

**If unsure, ask the user before deciding.** When the ask feels big — or you notice you'd struggle to fit everything under a single ticket's acceptance criteria — default to offering `plan epic`. A lone Epic shell is rarely the right answer; if the natural type for the ask is "Epic", that's itself a signal to use `plan epic` instead of `jira-ticket-creator`.

### Sequence {#cmd.5.2.sequence}

Every request that involves a change to the project must follow this sequence:

1. **Pick the entry point** — Small ask → `start jira-ticket-creator`. Big ask → `plan epic`. When uncertain, ask the user. No code, no file edits, no installs until a ticket exists.
2. **Create the ticket(s)** — Run the chosen entry-point skill to create the ticket (or Epic + children) in JIRA.
3. **Ask for human confirmation** — Present what was created and explicitly ask: _"Should I proceed with execution?"_ Wait for approval before continuing.
4. **Pick the right executor** — Route through the **jira-router** or use the appropriate direct trigger to select the correct executor skill for the ticket type.
5. **Verify `.spec` files** — Before implementation, check whether any `.spec` files need to be created or updated for the change. If yes, update `.spec` first.
6. **Implement the change** — Only now execute the actual code/file changes as defined by the ticket and specs.

**Human confirmation is required at every decision point.** When in doubt, ask — never assume.

## JIRA Skills {#cmd.6.jira-skills}

All JIRA skills live under `.claude/skills/jira/`. When a trigger command is used, read the corresponding `SKILL.md` and **immediately begin the flow defined inside it**.

### Smart Router (Primary Entry Point) {#cmd.6.1.smart-router}

**When the user mentions any JIRA ticket ID** (e.g., `implement ACD-18`, `execute ACD-5`, `ACD-18 give steps`, or just `ACD-18`), use the **jira-router** skill. It fetches the ticket, detects the issue type, and auto-routes to the correct executor:

| Issue Type          | Routes To               | Equivalent Command                |
| ------------------- | ------------------------ | --------------------------------- |
| **Epic**            | `jira-epic-orchestrator` | `execute epic <KEY>`              |
| **Story**           | `jira-story-executor`    | `execute story <KEY>`             |
| **Task**            | `jira-executor`          | `start jira-executor <KEY>`       |
| **Subtask**         | `jira-executor`          | `start jira-executor <KEY>`       |
| **Bug**             | `jira-bug-executor`      | `start jira-bug-executor <KEY>`   |

Router skill: [`.claude/skills/jira/jira-router/SKILL.md`](.claude/skills/jira/jira-router/SKILL.md)

### Direct Triggers {#cmd.6.2.direct-triggers}

These explicit commands still work and bypass the router:

| Trigger                                         | Skill File                                                                                                   | Description                                     |
| ----------------------------------------------- | ------------------------------------------------------------------------------------------------------------ | ----------------------------------------------- |
| `start jira-ticket-creator`                     | [`.claude/skills/jira/jira-ticket-creator/SKILL.md`](.claude/skills/jira/jira-ticket-creator/SKILL.md)       | Interactive JIRA ticket creation                |
| `start jira-executor <ticket-id>`               | [`.claude/skills/jira/jira-executor/SKILL.md`](.claude/skills/jira/jira-executor/SKILL.md)                   | Task/Subtask implementation                     |
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

## Registered Specs {#cmd.7.registered-specs}

| Spec ID       | File                                                         | Description                  |
| ------------- | ------------------------------------------------------------ | ---------------------------- |
| `naming-rule` | [`.spec/rules/naming-rules.md`](.spec/rules/naming-rules.md) | Naming conventions — folders |

## Registered Flows {#cmd.8.registered-flows}

No flows registered yet. Flows will be added as user-facing features are implemented.
