# JIRA-Driven Claude Code Approach for SaaS (Vibe Engineering)

> **JIRA Ticket:** [MD-11](https://vinay251101.atlassian.net/browse/MD-11)

## Overview

We follow a development approach that combines three pillars to build and maintain our SaaS platform. Each pillar has a clearly defined role and boundary.

## The Three Pillars

### 1. JIRA — What

JIRA is the single source of truth for **what** the team is building.

- Every feature, bug fix, and task originates as a JIRA ticket.
- Tickets capture the requirements, acceptance criteria, and priority.
- No work begins without a corresponding JIRA ticket.

### 2. Spec Config Markdown Library — How

The Spec Config Markdown Library (hierarchical `CLAUDE.md` indexes) is the single source of truth for **how the system works**.

- Covers contracts, schemas, rules, and architecture decisions.
- Organized as hierarchical `CLAUDE.md` files that serve as indexes at each level of the project.
- This root `CLAUDE.md` is the top-level entry point; module-level `CLAUDE.md` files provide deeper context as the project grows.

### 3. Claude Code — Disciplined Implementation Partner

Claude Code is a disciplined implementation partner that:

- **Writes code** based on requirements defined in JIRA tickets and specifications in the Markdown library.
- **Formats and organizes human-provided content** into JIRA tickets and MD files.
- **Never originates substance.** All decisions about *what* to build, *why* to build it, and *how* the system should behave come from humans via JIRA and the Spec Config Markdown Library. Claude Code executes — it does not decide.

## Boundaries

| Responsibility | Owner |
|---|---|
| Deciding what to build | Humans via **JIRA** |
| Defining how the system works (contracts, schemas, rules, architecture) | Humans via **Spec Config Markdown Library** |
| Writing code that implements the above | **Claude Code** |
| Formatting/organizing content into tickets and MD files | **Claude Code** |
| Originating new requirements, business logic, or architecture decisions | **Humans only** — Claude Code never originates substance |

## JIRA Skills

All JIRA skills live under `.claude/skills/jira/`. When a trigger command is used, read the corresponding `SKILL.md` below and **immediately begin the flow defined inside it**. Do NOT scan, read, or explore any other project files — go straight to Step 1 of the skill's flow and start human interaction.

**Trigger styles:**
- **`start <skill-name>`** — Used for `ticket-creator`, `executor`, `batch-executor`, and `flow-creator`
- **Direct commands** — Used for `epic-orchestrator` modes: `plan epic`, `orchestrate epic`, `execute epic <KEY>`, `update epic <KEY>`, `update and execute epic <KEY>`
- **`start jira-epic-orchestrator`** — Opens an interactive menu to choose which orchestrator mode to run

| Trigger | Skill File | Description |
|---|---|---|
| `start jira-ticket-creator` | [`.claude/skills/jira/jira-ticket-creator/SKILL.md`](.claude/skills/jira/jira-ticket-creator/SKILL.md) | Interactive JIRA ticket creation in the ACD (Aqua Claude Dev) space |
| `start jira-executor <ticket-id \| ticket-url>` | [`.claude/skills/jira/jira-executor/SKILL.md`](.claude/skills/jira/jira-executor/SKILL.md) | Step-by-step JIRA ticket implementation workflow with spec-first check, flow detection, and separate commits |
| `start flow-creator` | [`.claude/skills/jira/jira-flow-creator/SKILL.md`](.claude/skills/jira/jira-flow-creator/SKILL.md) | Standalone flow documentation — create/update flow.md, e2e.yaml, and unit.yaml |
| `plan epic` | [`.claude/skills/jira/jira-epic-orchestrator/SKILL.md`](.claude/skills/jira/jira-epic-orchestrator/SKILL.md) | Plan feature hierarchy and create JIRA tickets only — no code execution |
| `orchestrate epic` | [`.claude/skills/jira/jira-epic-orchestrator/SKILL.md`](.claude/skills/jira/jira-epic-orchestrator/SKILL.md) | Plan, create JIRA tickets, then immediately start executing pending tickets |
| `execute epic <EPIC-KEY>` | [`.claude/skills/jira/jira-epic-orchestrator/SKILL.md`](.claude/skills/jira/jira-epic-orchestrator/SKILL.md) | Execute an existing epic — implement only pending/unfinished tickets |
| `update epic <EPIC-KEY>` | [`.claude/skills/jira/jira-epic-orchestrator/SKILL.md`](.claude/skills/jira/jira-epic-orchestrator/SKILL.md) | Modify an existing epic's structure (add, remove, edit, restructure) — no execution |
| `update and execute epic <EPIC-KEY>` | [`.claude/skills/jira/jira-epic-orchestrator/SKILL.md`](.claude/skills/jira/jira-epic-orchestrator/SKILL.md) | Modify an existing epic, then execute pending tickets |
| `start jira-batch-executor <KEY-1, KEY-2, ...>` | [`.claude/skills/jira/jira-batch-executor/SKILL.md`](.claude/skills/jira/jira-batch-executor/SKILL.md) | Execute multiple standalone tickets in sequence with full git control after each ticket |

### How the skills relate

| Skill | Scope | Creates tickets? | Writes code? | Flow docs? | Git control |
|---|---|---|---|---|---|
| `ticket-creator` | Single ticket | Yes (1 ticket) | No | No | None |
| `executor` | Single ticket | No | Yes (implements 1 ticket) | Yes (auto-detect) | Separate commits (spec → flow → code) |
| `flow-creator` | Single flow | No | No | Yes (standalone) | Commit flow files |
| `batch-executor` | Multiple independent tickets | No | Yes (implements N tickets) | Yes (auto-detect) | Full menu per ticket with separate commits |
| `epic-orchestrator` | Epic / Initiative | Yes (batch) | Yes (in execute modes) | Yes (auto-detect in execute) | Separate commits per story |

### Shared protocols

All JIRA skills reference reusable protocol files in `.claude/skills/jira/_shared/references/`. These ensure consistent behavior across skills:

| Protocol | File | Used By |
|---|---|---|
| Human Confirmation | `human-confirmation-protocol.md` | All skills — enhanced confirmation with "add more details" option |
| Acceptance Criteria Checklist | `acceptance-criteria-checklist.md` | ticket-creator, epic-orchestrator (Mode 1/2/4) |
| Technical Deep-Dive | `technical-deep-dive.md` | epic-orchestrator (Mode 1/2/4 task decomposition) |
| Spec-First | `spec-first-protocol.md` | executor, batch-executor, epic-orchestrator (Mode 3/5) |
| Flow | `flow-protocol.md` | executor, batch-executor, epic-orchestrator (Mode 3/5), flow-creator |
| Implementation | `implementation-protocol.md` | executor, batch-executor, epic-orchestrator (Mode 3/5) |
| Git Workflow | `git-workflow.md` | executor (Level 1), epic-orchestrator (Level 2), batch-executor (Level 3) |

### Separate commit protocol

When a ticket involves changes across multiple categories, each category is committed separately in this order:

1. **Spec files** (`.spec/`) — `[KEY]: Add/Update spec — [description]`
2. **Flow files** (`.flows/`) — `[KEY]: Add/Update [flow-name] flow documentation`
3. **Implementation code** — `[KEY]: [Summary]`

This keeps git history clean and makes code review easier. If only one category has changes, a single commit is used.

### JIRA Epic Orchestrator — Complete Reference

The `epic-orchestrator` skill is the full-lifecycle counterpart to `ticket-creator` and `executor`. Where those skills work on single tickets, the orchestrator works at epic scale with the same discipline.

#### Mode summary

| # | Command | What it does | Creates tickets? | Writes code? |
|---|---|---|---|---|
| 1 | `plan epic` | Interactive planning → scope assessment → story/task breakdown with technical deep-dives → output docs → batch JIRA creation | Yes | No |
| 2 | `orchestrate epic` | Mode 1 (plan + create) then immediately flows into Mode 3 (execute pending) | Yes | Yes |
| 3 | `execute epic <KEY>` | Fetch existing epic → implement pending stories/tasks one by one → git commit per story → update JIRA statuses | No | Yes |
| 4 | `update epic <KEY>` | Load existing epic → add/remove/edit/restructure stories and tasks → update JIRA and spec file | Modifies existing | No |
| 5 | `update and execute epic <KEY>` | Mode 4 (update) then immediately flows into Mode 3 (execute pending) | Modifies existing | Yes |

#### Hierarchy scaling (auto-detected in Mode 1 & 2)

- **Small/medium scope** (1–4 capabilities, one functional area) → `Epic → Stories → Tasks`
- **Large scope** (3+ functional areas, 6+ capabilities) → `Initiative → Epics → Stories → Tasks`

The user can override the recommendation in either direction.

#### Output formats (Mode 1 & 2, user's choice)

- **Markdown** → `.spec/epics/<name>.md` or `.spec/initiatives/<name>.md`
- **Excel (.xlsx)** → Structured workbook with Overview, Stories, Tasks, Database Models, API Contracts sheets
- **Word (.docx)** → Professional document with title page, TOC, technical appendices

#### Spec file as source of truth

During planning (Mode 1/2), technical specs (database schemas, migration scripts, API contracts) are gathered interactively and saved to `.spec/`. During execution (Mode 3/5), this spec file is the primary reference — it takes priority over JIRA ticket descriptions.

### JIRA Batch Executor — Complete Reference

The `batch-executor` skill handles multiple standalone JIRA tickets in sequence. Unlike the orchestrator (which works on epic hierarchies), the batch executor treats each ticket as an independent work item and provides full git control between each.

#### Trigger

```
start jira-batch-executor ACD-1, ACD-2, ACD-3
```

Accepts comma-separated ticket keys from any JIRA project (ACD, etc.). Tickets do not need to belong to the same epic.

#### Execution flow (per ticket)

Each ticket follows the same disciplined flow as `executor`:

| Step | What happens | User confirms? |
|---|---|---|
| Fetch & understand | Fetches ticket, rephrases requirements, shows acceptance criteria | Yes |
| Transition | Moves ticket to In Progress | Auto |
| Spec-first check | Verifies `.spec` files exist, creates missing ones | Yes |
| Flow check | Detects user flows, creates flow documentation if applicable | Yes |
| Implementation plan | Shows code snippets and file changes | Yes |
| Implement | Writes the code | After confirmation |
| Git workflow | Full interactive git menu with separate commits (spec → flow → code) | Yes, per operation |
| JIRA update | Adds comment, optionally transitions to Done | Yes |

#### Git menu (after each ticket)

After every ticket is implemented, the user gets a full interactive git menu that loops until they choose to move on:

| Option | What it does |
|---|---|
| Commit changes | Stage + commit with auto-generated or custom message |
| Push to remote | Push current branch to origin |
| Create new branch | Create from current, main, or specified base branch |
| Switch branch | Switch with stash/commit warning for uncommitted changes |
| Merge branch | Merge another branch into current, with conflict resolution |
| Pull latest | Pull from remote with conflict handling |
| Create Pull Request | Create PR via `gh` CLI with title, target branch, description |
| Done, move to next | Exit git menu, proceed to next ticket |

#### When to use batch executor vs other skills

- **1 ticket** → use `executor`
- **Multiple unrelated tickets** → use `batch-executor`
- **All tickets in an epic (stories + tasks)** → use `epic-orchestrator` with `execute epic`

### Flow Creator — Complete Reference

The `flow-creator` skill documents user flows with full test coverage. It works both standalone and integrated into JIRA execution skills.

#### Trigger

```
start flow-creator
```

Also auto-triggered during `executor`, `batch-executor`, and `epic-orchestrator` execution when a ticket involves a user flow.

#### Output structure

```
.flows/
  <module>/
    <flow-name>/
      flow.md       — Step-by-step flow with conditions and all scenarios
      e2e.yaml      — Structured E2E test scenarios
      unit.yaml     — Unit test case specs by function/method
```

The `.flows/` folder mirrors the code's module structure (e.g., `.flows/auth/forgot-password/`, `.flows/billing/payment-checkout/`).

#### What it generates

- **flow.md** — Flow definition: actors, preconditions, numbered steps with decision points, happy path + error + edge case scenarios, business rules, postconditions
- **e2e.yaml** — E2E test scenarios: preconditions, step-by-step actions with expected results, postconditions, tags for categorization
- **unit.yaml** — Unit test specs: organized by function/method, test cases with inputs/expected outputs, mock requirements

#### When to use

- **Standalone** — `start flow-creator` to document any flow independently
- **During ticket execution** — Auto-detected when a ticket involves a user flow (login, registration, forgot-password, checkout, payment, onboarding, etc.)
- **Updating existing flows** — Load existing files, show diffs, confirm before overwriting

## Configure

### Spec Registration

Every markdown file created under the `.spec` folder must have a corresponding entry registered in this `CLAUDE.md` with a unique **ID key** for tracking.

#### ID Key Convention

- The ID key is derived from the spec file name by dropping the `.md` extension and removing any trailing plural "s" where appropriate (e.g., `naming-rules.md` → ID: `naming-rule`).
- Each section within a spec file gets a **scoped ID** formed as `<file-id>.<section>` (e.g., `naming-rule.folder` for the folder naming rules section).

### Registered Specs

| ID | File | Description |
|---|---|---|
| `naming-rule` | [`.spec/rules/naming-rules.md`](.spec/rules/naming-rules.md) | Naming conventions and rules for the project |

### Flow Registration

Every flow created under the `.flows` folder must have a corresponding entry registered here. The ID follows the pattern `<module>.<flow-name>`.

### Registered Flows

| ID | File | Description |
|---|---|---|
| — | — | No flows registered yet |
