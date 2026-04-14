<!-- ============================================================
     MODE 1 — PLAN EPIC
     ============================================================
     Trigger: plan epic
     
     This is the complete Mode 1 workflow for JIRA Epic Orchestrator.
     It runs through all 8 stages of planning:
     1. Feature Discovery
     2. Scope Assessment & Hierarchy Proposal
     3. Epic Definition(s)
     4. Story Breakdown
     5. Task Decomposition & Technical Deep-Dive
     6. Full Plan Review
     7. Output Generation
     8. Batch JIRA Creation
     
     This mode creates all JIRA tickets but does NOT execute them.
     After creation, the user can execute via "execute epic <KEY>".
     
     Reference protocols:
     - .claude/skills/jira/_shared/references/human-confirmation-protocol.md
     - .claude/skills/jira/_shared/references/acceptance-criteria-checklist.md
     - .claude/skills/jira/_shared/references/technical-deep-dive.md
     
     No code is written. Output includes markdown specs and JIRA tickets.
     ============================================================ -->

# Mode 1 — Plan Epic

**Trigger:** `plan epic`

**What it does:** Plan a complete feature hierarchy interactively and create all JIRA tickets. No code is executed.

**Key principle:** Follow Human Confirmation Protocol for every confirmation. One question at a time. Never originate substance — all decisions come from the user.

## Section IDs

**Prefix:** `plan`

When starting each stage, display the section ID:

```
▶ [plan.N.name] — Stage title
```

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

---

## Stage 1 — Feature Discovery {#plan.1.feature-discovery}

Ask these questions **one at a time** (plain text, not AskUserQuestion):

1. **"What feature are you planning? Give me a short name and one-sentence description."**
   
   Collect: Feature name, one-sentence description.

2. **"Who is this for? (e.g., admin users, end customers, internal teams)"**
   
   Collect: Target audience.

3. **"What's the business goal? Why are we building this?"**
   
   Collect: Business motivation.

4. **"List the major capabilities this feature needs. (e.g., user registration, payment processing, reporting dashboard)"**
   
   Collect: Comma-separated list of capabilities.

After collecting all four answers, move to Stage 2.

---

## Stage 2 — Scope Assessment & Hierarchy Proposal {#plan.2.scope-assessment}

Based on the capabilities listed in Stage 1, assess whether this should be an **Initiative** (multiple epics) or a **Single Epic**.

### Assessment Criteria

**Signals that it needs an Initiative (multiple epics):**
- Feature spans 3+ distinct functional areas
- Capabilities are loosely coupled and could ship independently
- 6+ major capabilities listed

**Signals that a single Epic is enough:**
- Feature focused on one functional area
- Capabilities are tightly coupled and ship together
- 1-4 major capabilities listed

### Present Assessment

Display a clear assessment block:

```
SCOPE ASSESSMENT

Feature: [Feature Name]
Capabilities: [Count] listed

Analysis:
- Functional areas identified: [List the areas]
- Coupling: [Tightly / Loosely coupled]
- Recommended hierarchy: [Initiative → Epics → Stories → Tasks] OR [Epic → Stories → Tasks]

Rationale: [Brief explanation based on signals above]
```

### Confirm with User

Use `AskUserQuestion`:

- **This assessment works for me** — Use the recommended hierarchy
- **Make it bigger (add more epics/scope)** — I want to expand the scope
- **Make it smaller (single epic)** — I want to narrow the focus
- **I want to add more context first** — There are details I should provide before we decide

**If user selects "add more context":**
   - Ask: "What additional context should I know?"
   - Collect the context and re-run the assessment.

After confirmation, move to Stage 3.

---

## Stage 3 — Epic Definition(s) {#plan.3.epic-definition}

### Single Epic Path

If the recommendation (or user override) is a **single Epic**:

Present a polished epic definition:

```
EPIC DEFINITION

Epic Name: [Name]

Summary:
[One sentence clearly stating what this epic delivers]

Description:
[2-3 sentences explaining the user-facing benefit and business context]

Goals:
- [Specific goal 1]
- [Specific goal 2]
- [Specific goal 3]

Capabilities Included:
- [Capability 1]
- [Capability 2]
- [Capability 3]
- ...
```

Use `AskUserQuestion`:

- **Looks good** — This definition is ready
- **Refine the wording** — I want to adjust the description or goals
- **Add more details** — I want to include additional information

**If "Refine the wording":**
   - Ask: "What would you like to change? (summary, description, goals, capabilities)"
   - Collect adjustments and re-present the updated definition.

**If "Add more details":**
   - Ask: "What should we add?"
   - Collect and incorporate.

Repeat until confirmed. Then move to Stage 4.

### Initiative Path

If the recommendation (or user override) is an **Initiative** with multiple epics:

For each epic, run this process one at a time:

Present a proposed epic definition:

```
EPIC DEFINITION — [Epic Name] (Epic 1 of N)

Summary:
[One sentence clearly stating what this epic delivers]

Description:
[2-3 sentences explaining the user-facing benefit]

Goals:
- [Specific goal 1]
- [Specific goal 2]
- [Specific goal 3]

Capabilities Included:
- [Capability 1]
- [Capability 2]
- ...
```

Use `AskUserQuestion`:

- **Looks good** — This epic definition is ready
- **Refine the wording** — I want to adjust the description or goals
- **Add more details** — I want to include additional information

Repeat the refinement loop until confirmed.

After the first epic is confirmed, ask:

**"Ready to define the next epic?"**

If yes, repeat the process for the next epic. Continue until all epics are defined.

After all epics are confirmed, move to Stage 4.

---

## Stage 4 — Story Breakdown (per Epic) {#plan.4.story-breakdown}

### For Each Epic (in order)

Present the list of proposed user stories:

```
USER STORIES — [Epic Name]

Based on the capabilities and goals, here are the proposed user stories:

1. As a [who], I want [what], so that [why]
   Priority: [High / Medium / Low]

2. As a [who], I want [what], so that [why]
   Priority: [High / Medium / Low]

3. As a [who], I want [what], so that [why]
   Priority: [High / Medium / Low]

...

Total: [X] stories
```

Use `AskUserQuestion`:

- **This story list looks good** — Proceed with these stories
- **Let me adjust** — I want to modify, remove, or reorder stories
- **Looks good but I want to add stories** — I have additional stories to include

**If "Let me adjust":**
   - Ask: "What changes would you like? (remove, reorder, modify priorities, change descriptions)"
   - Collect adjustments, update the list, and re-present.

**If "Add stories":**
   - Ask: "What additional stories?"
   - Collect and add, then re-present the full updated list.

Repeat until confirmed.

### Run Acceptance Criteria Checklist Protocol

After the story list is confirmed, run **Acceptance Criteria Checklist Protocol** for each story (read `.claude/skills/jira/_shared/references/acceptance-criteria-checklist.md`).

For each story:

1. Generate suggested acceptance criteria based on the story description
2. Present as a checklist
3. Let the user select, add, or modify
4. Confirm the final list

Store the confirmed criteria with the story for later JIRA ticket creation.

### Move to Next Epic

After all stories for this epic are defined and their acceptance criteria are locked in, move to the next epic (if Initiative).

When all epics have their stories and criteria, move to Stage 5.

---

## Stage 5 — Task Decomposition & Technical Deep-Dive (per Story) {#plan.5.task-decomposition}

### For Each Story (in order, across all epics)

#### Step 1 — Task Breakdown

Present the proposed task breakdown:

```
TASKS — [Story Name]

Story: As a [who], I want [what], so that [why]

Proposed Tasks:

1. [Task name] — [Description of what needs to be done]
2. [Task name] — [Description of what needs to be done]
3. [Task name] — [Description of what needs to be done]
...

Total: [X] tasks
```

Use `AskUserQuestion`:

- **This task list looks good** — Proceed with these tasks
- **Let me adjust** — I want to modify or add tasks
- **Add more tasks** — I have additional tasks to include

Handle adjustments per the Human Confirmation Protocol pattern. Repeat until confirmed.

#### Step 2 — Technical Deep-Dive for Each Task

After the task list is confirmed, run **Technical Deep-Dive Protocol** for each task (read `.claude/skills/jira/_shared/references/technical-deep-dive.md`).

For each task:

1. Identify what technical artifacts are needed (database work, API work, UI work, business logic, infrastructure, etc.)
2. Ask questions one at a time to gather specifications
3. Generate the technical artifacts (schemas, API contracts, UI specs, logic flows, etc.)
4. Present and confirm per Human Confirmation Protocol

Present the generated specs:

```
TECHNICAL SPECIFICATIONS — [Task Name]

[Generated based on deep-dive:
- Database models (if applicable)
- API contracts (if applicable)
- UI component specs (if applicable)
- Business logic flow (if applicable)
- Infrastructure specs (if applicable)]
```

Use `AskUserQuestion`:

- **Yes, specs are correct** — Save and move to next task
- **No, let me adjust** — I want to modify the technical details
- **Add more details to these specs** — There's more to include

Repeat for every task in every story until all specs are locked in.

Move to Stage 6.

---

## Stage 6 — Full Plan Review {#plan.6.full-plan-review}

Before creating JIRA tickets, insert a **confidence checkpoint** (per Human Confirmation Protocol):

```
Quick check before we finalize:

We've planned the complete feature hierarchy with all stories,
tasks, acceptance criteria, and technical specifications.
Next up: Create all JIRA tickets and generate output documents.

Everything looking good so far?
```

Use `AskUserQuestion`:

- **Yes, all good — finalize** — Everything is on track, create the tickets
- **Hold on, I want to revisit something** — Let me go back and review something
- **Yes, but I want to add something first** — There's additional context to include

**If "revisit something":**
   - Ask: "What would you like to review? (feature scope, epic definitions, stories, tasks, specs)"
   - Navigate back to that section and make adjustments.

**If "add something first":**
   - Ask: "What should we add?"
   - Incorporate and re-present the updated plan.

### Present Full Hierarchy

After confidence checkpoint passes, present the complete plan with ASCII hierarchy diagram:

**Single Epic Format:**

```
COMPLETE PLAN — [Feature Name]

Feature: [Name]
Business Goal: [Goal]
Target Audience: [Audience]

├── Epic: [Epic Name]
    ├── Story 1: As a [who], I want [what]
    │   ├── Task 1: [Task name]
    │   ├── Task 2: [Task name]
    │   └── Task 3: [Task name]
    ├── Story 2: As a [who], I want [what]
    │   ├── Task 1: [Task name]
    │   └── Task 2: [Task name]
    └── Story 3: As a [who], I want [what]
        └── Task 1: [Task name]

Total Stories: [X]
Total Tasks: [X]
```

**Initiative Format:**

```
COMPLETE PLAN — [Feature Name]

Feature: [Name]
Business Goal: [Goal]
Target Audience: [Audience]

├── Initiative: [Initiative Name]
    ├── Epic 1: [Epic Name]
    │   ├── Story 1: As a [who], I want [what]
    │   │   ├── Task 1: [Task name]
    │   │   └── Task 2: [Task name]
    │   └── Story 2: As a [who], I want [what]
    │       └── Task 1: [Task name]
    ├── Epic 2: [Epic Name]
    │   ├── Story 1: As a [who], I want [what]
    │   │   └── Task 1: [Task name]
    │   └── Story 2: As a [who], I want [what]
    │       └── Task 1: [Task name]
    └── Epic 3: [Epic Name]
        └── Story 1: As a [who], I want [what]
            └── Task 1: [Task name]

Total Epics: [X]
Total Stories: [X]
Total Tasks: [X]
```

Use `AskUserQuestion`:

- **This looks good — finalize** — I'm satisfied with the complete plan
- **I want to make changes** — I see something that needs adjustment
- **Looks good but I want to add more details** — I have additional details to include

**If changes requested:**
   - Ask: "What changes would you like?"
   - Navigate back to the relevant section and update.

Repeat until the user confirms the plan is final. Move to Stage 7.

---

## Stage 7 — Output Generation {#plan.7.output-generation}

Ask: **"What output formats would you like?"**

Use `AskUserQuestion` (multiselect):

- **Markdown spec file** — Save to `.spec/epics/` or `.spec/initiatives/`
- **Excel (.xlsx) workbook** — Professional spreadsheet with structured sheets
- **Word (.docx) document** — Professional document with title page and TOC
- **All formats** — Generate all three

### Generate Markdown

**File path:** `.spec/epics/<feature-name>.md` (single epic) or `.spec/initiatives/<feature-name>.md` (initiative)

**Content structure:**

```
# [Feature Name]

## Overview

**Business Goal:** [Goal]
**Target Audience:** [Audience]
**Capabilities:** [List]

## [Epic Name] (or Initiative > Epic 1, Epic 2, ...)

### Story 1: [Story title]

As a [who], I want [what], so that [why]

**Acceptance Criteria:**
- [Criterion 1]
- [Criterion 2]
- ...

#### Task 1: [Task name]

[Task description]

**Technical Specifications:**
[Database schemas, API contracts, UI specs, etc. from deep-dive]

#### Task 2: [Task name]

...

### Story 2: [Story title]

...

## Database Models

[All database schemas from all tasks]

## API Contracts

[All API endpoint specs from all tasks]

## Business Rules

[All business logic from all tasks]
```

Save the file and register it in the project's `CLAUDE.md` under the `Registered Specs` section.

### Generate Excel

**File path:** `.spec/epics/<feature-name>.xlsx` or `.spec/initiatives/<feature-name>.xlsx`

**Sheets:**
1. **Overview** — Feature name, business goal, audience, total stories/tasks, date created
2. **Epics** (if Initiative) — Epic names, summaries, goals, included stories count
3. **Stories** — Story ID, title, description, acceptance criteria count, status
4. **Tasks** — Task ID, title, description, parent story, status
5. **Database Models** — All schemas with columns, types, constraints
6. **API Contracts** — All endpoints with methods, paths, request/response schemas

### Generate Word

**File path:** `.spec/epics/<feature-name>.docx` or `.spec/initiatives/<feature-name>.docx`

**Structure:**
1. Title page — Feature name, date, business goal
2. Table of Contents
3. Executive Summary — Overview and business case
4. Feature Breakdown — Epic(s), stories, tasks hierarchy
5. Acceptance Criteria — All stories and their criteria
6. Technical Specifications — Database models, API contracts, UI specs
7. Appendix — Full task details

---

## Stage 8 — Batch JIRA Creation {#plan.8.batch-jira-creation}

### Confirmation Before Creating Tickets

Present the JIRA creation plan:

```
JIRA TICKET CREATION PLAN

Project: ACD (Aqua Claude Dev)
Target JIRA instance: [URL]

This will create:

[Single Epic path]
- 1 Epic
- [X] Stories
- [X] Tasks
Total: [X] tickets

[Initiative path]
- 1 Initiative
- [X] Epics
- [X] Stories
- [X] Tasks
Total: [X] tickets

All tickets will be linked in a hierarchy and populated with
acceptance criteria and technical specifications.
```

Use `AskUserQuestion`:

- **Yes, go ahead and create** — I confirm, create all tickets
- **Wait, let me review first** — I want to double-check something
- **No, skip this** — Don't create the tickets yet

**If "review first":**
   - Ask: "What would you like to review?"
   - Navigate back and show details.

**If "skip":**
   - End the mode. The plan exists but tickets are not created.
   - Offer: "You can create these tickets later with `start jira-executor` or re-run `plan epic`."

### Create JIRA Tickets

After user confirms, begin creation in this order:

**Single Epic Path:**
1. Create the Epic
2. Create each Story (linked as child of Epic)
3. Create each Task (linked as child of Story)

**Initiative Path:**
1. Create the Initiative
2. For each Epic:
   - Create the Epic (linked as child of Initiative)
   - Create each Story (linked as child of Epic)
   - Create each Task (linked as child of Story)

For each ticket created:
- Populate summary from story/task name
- Populate description with full text
- Add acceptance criteria as description sections
- Add technical specs as code blocks or attachments
- Set initial status to "Backlog" or "Ready"

### Confidence Checkpoint After First Ticket

After the first ticket is created successfully:

```
Quick check after first ticket:

Created [Ticket Key]: [Summary]

Does this look right? Are the story format, acceptance criteria,
and technical specs showing up correctly in JIRA?
```

Use `AskUserQuestion`:

- **Yes, looks good — continue** — Everything is correct, create the rest
- **Something looks wrong** — I see an issue with the format or content
- **Yes, but I want to adjust future tickets** — I want to modify how the rest are created

**If issue found:**
   - Ask: "What needs to be fixed?"
   - Adjust and resume creation.

**If want adjustments:**
   - Ask: "What should change in the remaining tickets?"
   - Collect adjustments and continue.

Continue creating remaining tickets without stopping.

### Present Final Tree View

After all tickets are created, present the final hierarchy as a tree:

**Single Epic Format:**

```
JIRA TICKETS CREATED

ACD-100 [Epic] Feature Name
├── ACD-101 [Story] Story 1 title
│   ├── ACD-105 [Task] Task 1.1
│   ├── ACD-106 [Task] Task 1.2
│   └── ACD-107 [Task] Task 1.3
├── ACD-102 [Story] Story 2 title
│   ├── ACD-108 [Task] Task 2.1
│   └── ACD-109 [Task] Task 2.2
└── ACD-103 [Story] Story 3 title
    └── ACD-110 [Task] Task 3.1

Total: 10 tickets created
```

**Initiative Format:**

```
JIRA TICKETS CREATED

ACD-100 [Initiative] Feature Name
├── ACD-101 [Epic] Epic 1 name
│   ├── ACD-105 [Story] Story 1.1 title
│   │   ├── ACD-111 [Task] Task 1.1.1
│   │   └── ACD-112 [Task] Task 1.1.2
│   └── ACD-106 [Story] Story 1.2 title
│       └── ACD-113 [Task] Task 1.2.1
├── ACD-102 [Epic] Epic 2 name
│   ├── ACD-107 [Story] Story 2.1 title
│   │   └── ACD-114 [Task] Task 2.1.1
│   └── ACD-108 [Story] Story 2.2 title
│       └── ACD-115 [Task] Task 2.2.1
└── ACD-103 [Epic] Epic 3 name
    └── ACD-109 [Story] Story 3.1 title
        └── ACD-116 [Task] Task 3.1.1

Total: 16 tickets created
```

### Add Comment to Top-Level Ticket

Add a comment to the Epic (single epic) or Initiative (initiative) ticket with:

```
Planning completed via Mode 1 (Plan Epic).

Hierarchy:
- [X] Stories
- [X] Tasks
- All acceptance criteria and technical specifications populated

Next step: Execute with "execute epic <EPIC-KEY>" or "execute epic <INITIATIVE-KEY>"
```

### Final Confirmation

Present final summary:

```
PLANNING COMPLETE

All [X] JIRA tickets have been created in the ACD project.

Epic Key: [ACD-XXX]

You can now:
- Review tickets in JIRA at [URL]
- Execute with: execute epic ACD-XXX
- Modify the epic with: update epic ACD-XXX
- Continue planning a new feature with: plan epic
```

---

## Mode 1 Ends Here

No code is written. All JIRA tickets are created and ready for execution via `execute epic <KEY>`.
