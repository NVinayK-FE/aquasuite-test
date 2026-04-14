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

When a trigger command is used, read the corresponding `SKILL.md` below and **immediately begin the flow defined inside it**. Do NOT scan, read, or explore any other project files — go straight to Step 1 of the skill's flow and start human interaction.

**Trigger styles:**
- **`start <skill-name>`** — Used for `jira-ticket-creator`, `jira-markdown-creator`, `jira-executor`, and `jira-batch-executor`
- **Direct commands** — Used for `jira-epic-orchestrator` modes: `plan epic`, `orchestrate epic`, `execute epic <KEY>`, `update epic <KEY>`, `update and execute epic <KEY>`
- **`start jira-epic-orchestrator`** — Opens an interactive menu to choose which orchestrator mode to run

| Trigger | Skill File | Description |
|---|---|---|
| `start jira-ticket-creator` | [`.claude/skills/jira-ticket-creator/SKILL.md`](.claude/skills/jira-ticket-creator/SKILL.md) | Interactive JIRA ticket creation in the ACD (Aqua Claude Dev) space |
| `start jira-markdown-creator` | [`.claude/skills/jira-markdown-creator/SKILL.md`](.claude/skills/jira-markdown-creator/SKILL.md) | Interactive JIRA ticket creation in the MD (Aqua Claude Docs) space |
| `start jira-executor <ticket-id \| ticket-url>` | [`.claude/skills/jira-executor/SKILL.md`](.claude/skills/jira-executor/SKILL.md) | Step-by-step JIRA ticket implementation workflow (e.g., `start jira-executor ACD-4` or `start jira-executor https://vinay251101.atlassian.net/browse/ACD-4`) |
| `plan epic` | [`.claude/skills/jira-epic-orchestrator/SKILL.md`](.claude/skills/jira-epic-orchestrator/SKILL.md) | Plan feature hierarchy and create JIRA tickets only — no code execution |
| `orchestrate epic` | [`.claude/skills/jira-epic-orchestrator/SKILL.md`](.claude/skills/jira-epic-orchestrator/SKILL.md) | Plan, create JIRA tickets, then immediately start executing pending tickets |
| `execute epic <EPIC-KEY>` | [`.claude/skills/jira-epic-orchestrator/SKILL.md`](.claude/skills/jira-epic-orchestrator/SKILL.md) | Execute an existing epic — implement only pending/unfinished tickets |
| `update epic <EPIC-KEY>` | [`.claude/skills/jira-epic-orchestrator/SKILL.md`](.claude/skills/jira-epic-orchestrator/SKILL.md) | Modify an existing epic's structure (add, remove, edit, restructure) — no execution |
| `update and execute epic <EPIC-KEY>` | [`.claude/skills/jira-epic-orchestrator/SKILL.md`](.claude/skills/jira-epic-orchestrator/SKILL.md) | Modify an existing epic, then execute pending tickets |
| `start jira-batch-executor <KEY-1, KEY-2, ...>` | [`.claude/skills/jira-batch-executor/SKILL.md`](.claude/skills/jira-batch-executor/SKILL.md) | Execute multiple standalone tickets in sequence with full git control (commit, push, branch, merge, pull, PR) after each ticket |

### JIRA Epic Orchestrator — Complete Reference

The `jira-epic-orchestrator` skill is the full-lifecycle counterpart to `jira-ticket-creator` and `jira-executor`. Where those skills work on single tickets, the orchestrator works at epic scale with the same discipline.

#### How the skills relate

| Skill | Scope | Creates tickets? | Writes code? | Git control |
|---|---|---|---|---|
| `jira-ticket-creator` | Single ticket | Yes (1 ticket) | No | None |
| `jira-markdown-creator` | Single ticket | Yes (1 ticket, MD space) | No | None |
| `jira-executor` | Single ticket | No | Yes (implements 1 ticket) | Commit + push |
| `jira-batch-executor` | Multiple independent tickets | No | Yes (implements N tickets) | Full menu (commit, push, branch, merge, pull, PR) per ticket |
| `jira-epic-orchestrator` | Epic / Initiative | Yes (batch) | Yes (in execute modes) | Commit + push per story |

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

The `jira-batch-executor` skill handles multiple standalone JIRA tickets in sequence. Unlike the orchestrator (which works on epic hierarchies), the batch executor treats each ticket as an independent work item and provides full git control between each.

#### Trigger

```
start jira-batch-executor ACD-1, ACD-2, ACD-3
```

Accepts comma-separated ticket keys from any JIRA project (ACD, MD, etc.). Tickets do not need to belong to the same epic.

#### Execution flow (per ticket)

Each ticket follows the same disciplined flow as `jira-executor`:

| Step | What happens | User confirms? |
|---|---|---|
| Fetch & understand | Fetches ticket, rephrases requirements, shows acceptance criteria | Yes |
| Transition | Moves ticket to In Progress | Auto |
| Scan specs | Finds relevant `.spec` files and sections | Yes |
| Implementation plan | Shows code snippets and file changes | Yes |
| Implement | Writes the code | After confirmation |
| Git workflow | Full interactive git menu (see below) | Yes, per operation |
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

- **1 ticket** → use `jira-executor`
- **Multiple unrelated tickets** → use `jira-batch-executor`
- **All tickets in an epic (stories + tasks)** → use `jira-epic-orchestrator` with `execute epic`

## Configure

Every markdown file created under the `.spec` folder must have a corresponding entry registered in this `CLAUDE.md` with a unique **ID key** for tracking.

### ID Key Convention

- The ID key is derived from the spec file name by dropping the `.md` extension and removing any trailing plural "s" where appropriate (e.g., `naming-rules.md` → ID: `naming-rule`).
- Each section within a spec file gets a **scoped ID** formed as `<file-id>.<section>` (e.g., `naming-rule.folder` for the folder naming rules section).

### Registered Specs

| ID | File | Description |
|---|---|---|
| `naming-rule` | [`.spec/rules/naming-rules.md`](.spec/rules/naming-rules.md) | Naming conventions and rules for the project |
