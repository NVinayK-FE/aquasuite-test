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
- **Never originates substance.** All decisions about _what_ to build, _why_ to build it, and _how_ the system should behave come from humans via JIRA and the Spec Config Markdown Library. Claude Code executes — it does not decide.

## Boundaries

| Responsibility                                                          | Owner                                                    |
| ----------------------------------------------------------------------- | -------------------------------------------------------- |
| Deciding what to build                                                  | Humans via **JIRA**                                      |
| Defining how the system works (contracts, schemas, rules, architecture) | Humans via **Spec Config Markdown Library**              |
| Writing code that implements the above                                  | **Claude Code**                                          |
| Formatting/organizing content into tickets and MD files                 | **Claude Code**                                          |
| Originating new requirements, business logic, or architecture decisions | **Humans only** — Claude Code never originates substance |

## JIRA Skills

All JIRA skills live under `.claude/skills/jira/`. When a trigger command is used, read the corresponding `SKILL.md` and **immediately begin the flow defined inside it**.

| Trigger                                         | Skill File                                                                                                   | Description                                     |
| ----------------------------------------------- | ------------------------------------------------------------------------------------------------------------ | ----------------------------------------------- |
| `start jira-ticket-creator`                     | [`.claude/skills/jira/jira-ticket-creator/SKILL.md`](.claude/skills/jira/jira-ticket-creator/SKILL.md)       | Interactive JIRA ticket creation                |
| `start jira-executor <ticket-id>`               | [`.claude/skills/jira/jira-executor/SKILL.md`](.claude/skills/jira/jira-executor/SKILL.md)                   | Single ticket implementation                    |
| `execute story <STORY-KEY>`                     | [`.claude/skills/jira/jira-story-executor/SKILL.md`](.claude/skills/jira/jira-story-executor/SKILL.md)       | Execute a story and all its subtasks            |
| `start jira-story-executor <STORY-KEY>`         | [`.claude/skills/jira/jira-story-executor/SKILL.md`](.claude/skills/jira/jira-story-executor/SKILL.md)       | Same as `execute story` — alternate trigger     |
| `start jira-batch-executor <KEY-1, KEY-2, ...>` | [`.claude/skills/jira/jira-batch-executor/SKILL.md`](.claude/skills/jira/jira-batch-executor/SKILL.md)       | Execute multiple standalone tickets in sequence |
| `plan epic`                                     | [`.claude/skills/jira/jira-epic-orchestrator/SKILL.md`](.claude/skills/jira/jira-epic-orchestrator/SKILL.md) | Plan and create JIRA tickets — no code          |
| `orchestrate epic`                              | [`.claude/skills/jira/jira-epic-orchestrator/SKILL.md`](.claude/skills/jira/jira-epic-orchestrator/SKILL.md) | Plan, create tickets, then execute              |
| `execute epic <EPIC-KEY>`                       | [`.claude/skills/jira/jira-epic-orchestrator/SKILL.md`](.claude/skills/jira/jira-epic-orchestrator/SKILL.md) | Execute pending stories/tasks in an epic        |
| `update epic <EPIC-KEY>`                        | [`.claude/skills/jira/jira-epic-orchestrator/SKILL.md`](.claude/skills/jira/jira-epic-orchestrator/SKILL.md) | Modify epic structure — no execution            |
| `update and execute epic <EPIC-KEY>`            | [`.claude/skills/jira/jira-epic-orchestrator/SKILL.md`](.claude/skills/jira/jira-epic-orchestrator/SKILL.md) | Modify epic, then execute pending tickets       |
| `start flow-creator`                            | [`.claude/skills/jira/jira-flow-creator/SKILL.md`](.claude/skills/jira/jira-flow-creator/SKILL.md)           | Standalone flow documentation                   |

## Registered Specs

| Spec ID       | File                                                         | Description                  |
| ------------- | ------------------------------------------------------------ | ---------------------------- |
| `naming-rule` | [`.spec/rules/naming-rules.md`](.spec/rules/naming-rules.md) | Naming conventions — folders |

## Registered Flows

No flows registered yet. Flows will be added as user-facing features are implemented.
