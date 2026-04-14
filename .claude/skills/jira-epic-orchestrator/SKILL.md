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
     JIRA EPIC ORCHESTRATOR
     ============================================================
     Full-lifecycle skill with 5 modes:
     
       Mode 1: plan epic              → Plan & create JIRA tickets only
       Mode 2: orchestrate epic       → Plan, create & immediately execute
       Mode 3: execute epic           → Execute an existing epic (pending only)
       Mode 4: update epic            → Modify existing epic structure, no execution
       Mode 5: update and execute epic → Modify then execute pending tickets
     
     Hierarchy (auto-detected by scope):
       Small/medium → Epic → Stories → Tasks
       Large        → Initiative → Epics → Stories → Tasks
     
     Target JIRA project: ACD (Aqua Claude Dev)
     
     Spec output locations:
       Single epic specs    → .spec/epics/<name>.md
       Initiative specs     → .spec/initiatives/<name>.md
     
     Document outputs (user's choice, any or all):
       - Markdown (.md)  → saved to .spec/ folder
       - Excel (.xlsx)   → structured summary with multiple sheets
       - Word (.docx)    → professional formatted document
     ============================================================ -->

# JIRA Epic Orchestrator

A full-lifecycle skill for planning, creating, executing, and modifying JIRA epics with deep technical specifications. It operates in 5 modes, each with its own trigger command.

**Project:** ACD (Aqua Claude Dev)

## Commands & Mode Detection

Each mode has a specific trigger command:

| Command | Mode | What It Does |
|---|---|---|
| `plan epic` | Mode 1 | Plan the feature hierarchy and create JIRA tickets. No code execution. |
| `orchestrate epic` | Mode 2 | Plan, create JIRA tickets, then immediately start executing pending tickets. |
| `execute epic <EPIC-KEY>` | Mode 3 | Execute an existing epic — implement only pending/unfinished tickets. |
| `update epic <EPIC-KEY>` | Mode 4 | Modify an existing epic's structure (add, remove, edit, restructure). No execution. |
| `update and execute epic <EPIC-KEY>` | Mode 5 | Modify an existing epic, then execute pending tickets. |

If the user says `start jira-epic-orchestrator` without a specific command, ask using `AskUserQuestion`:

- **Plan epic** — Plan a new feature and create JIRA tickets (no execution)
- **Orchestrate epic** — Plan, create JIRA tickets, and start executing immediately
- **Execute epic** — Implement an existing epic (pending tickets only)
- **Update epic** — Modify an existing epic's structure (no execution)

---

<!-- ============================================================
     MODE 1 — PLAN EPIC
     ============================================================
     Trigger: `plan epic`
     
     Interactive planning flow → JIRA ticket creation only.
     NO code is written. NO implementation happens.
     
     8 stages:
       1. Feature Discovery     → Collect high-level requirements
       2. Scope Assessment      → Decide Initiative vs Single Epic
       3. Epic Definition(s)    → Define each epic in detail
       4. Story Breakdown       → User stories per epic
       5. Task Decomposition    → Tasks + technical deep-dive per story
       6. Full Plan Review      → Present complete hierarchy for sign-off
       7. Output Generation     → Save as markdown / xlsx / docx
       8. Batch JIRA Creation   → Create all tickets with proper linking
     
     Every stage gates on human confirmation before proceeding.
     
     This flow works exactly like jira-ticket-creator but at scale:
       - jira-ticket-creator creates ONE ticket at a time
       - plan epic creates an ENTIRE hierarchy at once
       - Both ask one question at a time, rephrase for clarity,
         and confirm before creating anything in JIRA
     ============================================================ -->

## Mode 1 — Plan Epic

**Trigger:** `plan epic`

Plan a complete feature hierarchy interactively and create all JIRA tickets. No code is written — this is planning and ticket creation only. Works like `jira-ticket-creator` but at scale: instead of one ticket, you get an entire hierarchy.

Every stage requires human confirmation before proceeding. Never skip stages or combine them.

### Stage 1 — Feature Discovery

<!-- Collect the raw feature idea. Ask one question at a time,
     just like jira-ticket-creator asks for summary, then description,
     then acceptance criteria — never batch open-ended questions. -->

Ask the user to describe the feature at a high level. Use plain text messages (not AskUserQuestion) for open-ended questions. Ask these one at a time, waiting for a response before the next:

1. "What feature are you planning? Give me a short name and a sentence or two about what it does."
2. "Who is this for? (end users, internal team, API consumers, etc.)"
3. "What's the business goal or problem this solves?"
4. "Can you list the major capabilities or areas this feature covers?"

Wait for all answers before proceeding.

### Stage 2 — Scope Assessment & Hierarchy Proposal

<!-- Evaluate scope and recommend the right hierarchy level.
     This is unique to the orchestrator — jira-ticket-creator
     doesn't need this because it only creates single tickets.
     
     Decision criteria:
       Initiative → 3+ distinct functional areas, 6+ capabilities,
                    loosely coupled, multi-team/multi-sprint
       Single Epic → 1 functional area, 1-4 capabilities,
                     tightly coupled, single body of work
     
     The user can override the recommendation in either direction. -->

Based on the user's answers, assess the scope and propose the right hierarchy:

**Signals that it needs an Initiative (multiple epics):**
- The feature spans 3+ distinct functional areas (e.g., user management + billing + notifications + reporting)
- It would require multiple teams or multiple sprints across different domains
- The capabilities described are loosely coupled and could ship independently
- The user listed 6+ major capabilities

**Signals that a single Epic is enough:**
- The feature is focused on one functional area (e.g., user registration and authentication)
- All the capabilities are tightly coupled and depend on each other
- It could reasonably be planned and tracked as one body of work
- The user listed 1-4 major capabilities

Present your assessment clearly:

**If Initiative-level (large scope):**

```
Scope Assessment: INITIATIVE (multiple epics)

Your feature is broad enough that I'd recommend structuring it as an
Initiative with separate Epics for each major area. This keeps each
epic focused and independently deliverable.

Initiative: [title]
Description: [what this initiative achieves overall]

Proposed Epics:
1. [Epic 1 title] — [one-line scope description]
2. [Epic 2 title] — [one-line scope description]
3. [Epic 3 title] — [one-line scope description]
...
```

**If Epic-level (small/medium scope):**

```
Scope Assessment: SINGLE EPIC

Your feature is focused enough to be a single Epic with stories
and tasks underneath. No need for an Initiative wrapper.

Epic: [title]
Description: [professional rephrasing of what the feature does and why]
Target Users: [who benefits]
Business Goal: [why this matters]
```

Use `AskUserQuestion` to confirm:

- **Yes, this structure works** — Proceed with this hierarchy
- **No, I think it should be bigger** — Upgrade to Initiative with multiple epics
- **No, I think it should be smaller** — Downgrade to a single epic

Repeat until the user confirms the hierarchy level.

### Stage 3 — Epic Definition(s)

<!-- Define each epic in detail.
     - Initiative path: define each epic one at a time, confirm each
     - Single Epic path: define the one epic
     Similar to how jira-ticket-creator collects summary + description,
     but here we do it for each epic in the hierarchy. -->

**If Initiative path:** For each proposed epic under the initiative, one at a time, define it with the user:

```
Epic [N] of [total]: [Epic title]

Summary:       [concise title]
Description:   [what this epic covers]
Goal:          [what success looks like for this epic specifically]
```

Use `AskUserQuestion` to confirm each epic before moving to the next:

- **Yes, looks good** — Move to next epic definition
- **No, let me refine** — I want to adjust this epic

After all epics are defined, present the full Initiative overview and confirm before proceeding.

**If Single Epic path:** Present the polished epic definition:

```
Epic Summary:    [concise title, 5-15 words]
Description:     [professional rephrasing]
Target Users:    [who benefits]
Business Goal:   [why this matters]
```

Use `AskUserQuestion` to confirm:

- **Yes, looks good** — Move to story breakdown
- **No, let me refine** — I want to adjust something

### Stage 4 — Story Breakdown (per Epic)

<!-- Break each epic into user stories.
     Each story follows "As a [who], I want [what], so that [why]".
     If on Initiative path, repeat for each epic.
     Like jira-ticket-creator's description step but structured
     as user stories with priorities. -->

For each epic (or the single epic), propose user stories. Each story should represent a distinct user-facing or system capability.

```
Epic: [Epic title]

Proposed User Stories:

1. [Story title]
   As a [who], I want [what], so that [why].
   Priority: [Highest / High / Medium / Low]

2. [Story title]
   As a [who], I want [what], so that [why].
   Priority: [Highest / High / Medium / Low]

...
```

Use `AskUserQuestion` to confirm:

- **Yes, this breakdown works** — Move to task decomposition
- **No, let me adjust** — I want to add, remove, or modify stories

If adjusting, collect feedback, revise, and re-present. Repeat until confirmed.

If on the Initiative path, repeat Stage 4 for each epic before moving to Stage 5.

### Stage 5 — Task Decomposition & Technical Deep-Dive (per Story)

<!-- This is the heart of the orchestrator.
     For each story → break into tasks → deep-dive each task.
     
     The deep-dive goes beyond what jira-ticket-creator does.
     jira-ticket-creator collects a description and acceptance criteria.
     Here, we gather actual implementation artifacts:
       - Database schemas, migration scripts
       - API contracts, validation rules
       - UI component hierarchies
       - Business logic flows
       - Infrastructure specs
     
     These specs become part of the JIRA ticket description AND
     the spec file, so Mode 3 (execute) has everything ready. -->

For each confirmed story, one at a time, break it down into implementation tasks. Present the task list first:

```
Story: [Story title]

Tasks:

1. [Task title]
   Action:      [what needs to be done]
   Priority:    [Highest / High / Medium / Low]

2. [Task title]
   Action:      [what needs to be done]
   Priority:    [Highest / High / Medium / Low]
```

Use `AskUserQuestion` to confirm the task list:

- **Yes, these tasks are right** — Move to technical deep-dive
- **No, let me adjust** — I want to add, remove, or modify tasks

After task list is confirmed, go into a **technical deep-dive for each task**. This is the heart of the orchestrator — gather real implementation details interactively.

#### Technical Deep-Dive Protocol

For each task, identify what technical artifacts are needed and ask the user about them one question at a time:

**Database work** (data storage, models, entities):
- Ask: "What database engine? (Postgres, MySQL, MongoDB, etc.)"
- Ask: "What tables/collections are needed? For each one, what columns/fields, types, and constraints?"
- Ask: "Any foreign keys, indexes, or relationships between tables?"
- Ask: "Any seed data or default values needed?"
- Then generate and present:
  - Database model (entity definitions with types, constraints, relationships)
  - Migration/upgrade script (CREATE TABLE, ALTER TABLE, etc.)
  - Entity relationship diagram (as text)

**API work** (endpoints, services, integrations):
- Ask: "What endpoints are needed? (method, path, purpose)"
- Ask: "What are the request/response payloads? (fields, types, validations)"
- Ask: "Authentication/authorization requirements?"
- Ask: "Rate limiting, pagination, or versioning needs?"
- Then generate and present:
  - API contract (endpoint specs with request/response schemas)
  - Validation rules
  - Error response definitions

**UI work** (frontend, screens, components):
- Ask: "What screens or components are needed?"
- Ask: "What fields, interactions, and validations?"
- Ask: "Any design requirements or component library to follow?"
- Then generate and present:
  - Component hierarchy
  - State management needs
  - Key interactions and user flows

**Business logic** (rules, calculations, workflows):
- Ask: "What are the rules or conditions?"
- Ask: "Any edge cases or error scenarios?"
- Then generate and present:
  - Logic flow or decision tree
  - Error handling approach

**Infrastructure / DevOps** (deployment, config, CI/CD):
- Ask: "What environment or platform? (AWS, GCP, Docker, K8s, etc.)"
- Ask: "Any config, secrets, or environment variables needed?"
- Then generate and present:
  - Infrastructure spec
  - Config templates

After generating the technical specs for each task, present them and use `AskUserQuestion`:

- **Yes, specs are correct** — Move to next task
- **No, let me adjust** — I want to modify the technical details

Repeat for every task in the story, then move to the next story. Continue until all stories across all epics are fully specified.

### Stage 6 — Full Plan Review

<!-- Present the entire hierarchy in one consolidated view.
     Last chance to make changes before outputs and JIRA creation.
     Like jira-ticket-creator's Step 6 (review & confirm) but
     for the full hierarchy instead of a single ticket. -->

After everything is specified, present the complete hierarchy in one view.

**If Initiative-level:**

```
═══════════════════════════════════════════════════════
INITIATIVE: [Initiative title]
═══════════════════════════════════════════════════════

Description: [initiative description]

╔═══════════════════════════════════════════════════╗
║ EPIC 1: [Epic title]                              ║
╚═══════════════════════════════════════════════════╝

  ───────────────────────────────────────────────────
  STORY 1.1: [Story title]
    As a [who], I want [what], so that [why]
    Priority: [priority]
  ───────────────────────────────────────────────────

    TASK 1.1.1: [Task title]
      Priority: [priority]
      Technical Specs: [summary of DB models, API contracts, etc.]

    TASK 1.1.2: [Task title]
      Priority: [priority]
      Technical Specs: [summary]

╔═══════════════════════════════════════════════════╗
║ EPIC 2: [Epic title]                              ║
╚═══════════════════════════════════════════════════╝
  ...

═══════════════════════════════════════════════════════
TOTALS: [X] epics, [Y] stories, [Z] tasks
═══════════════════════════════════════════════════════
```

**If Single Epic:**

```
═══════════════════════════════════════════════════════
EPIC: [Epic title]
═══════════════════════════════════════════════════════

Description: [epic description]
Business Goal: [business goal]

───────────────────────────────────────────────────
STORY 1: [Story title]
  As a [who], I want [what], so that [why]
  Priority: [priority]
───────────────────────────────────────────────────

  TASK 1.1: [Task title] — [Technical Specs summary]
  TASK 1.2: [Task title] — [Technical Specs summary]

═══════════════════════════════════════════════════════
TOTAL: [X] stories, [Y] tasks
═══════════════════════════════════════════════════════
```

Use `AskUserQuestion` to confirm:

- **Yes, finalize this plan** — Move to output generation
- **No, I need changes** — Let me make final adjustments

### Stage 7 — Output Generation

<!-- Save the plan as documents. User picks formats.
     Like jira-ticket-creator's final step but produces
     full documents instead of a single JIRA ticket.
     
     File locations:
       Markdown → .spec/epics/ or .spec/initiatives/
       Excel/Word → project root or user-specified location
     
     After saving markdown, register it in root CLAUDE.md. -->

After the plan is finalized, ask what outputs the user wants.

Use `AskUserQuestion` (multiSelect: true):

- **Markdown spec** — Save as .md in the .spec/ folder (becomes part of your spec library)
- **Excel summary** — Generate an .xlsx with all tickets in a structured table
- **Word document** — Generate a polished .docx version of the full plan
- **All of the above** — Generate all three formats

Generate the selected outputs:

**Markdown:** Save to `.spec/epics/<epic-name>.md` (or `.spec/initiatives/<initiative-name>.md` if Initiative-level) with the complete hierarchy including all technical specs. Register it in the root `CLAUDE.md` under Registered Specs.

**Excel:** Create an `.xlsx` file with sheets:
- "Overview" — Initiative/Epic summary, business goal, counts
- "Epics" — All epics with descriptions (only if Initiative-level)
- "Stories" — All stories with parent epic, descriptions, priorities, acceptance criteria
- "Tasks" — All tasks with parent story, technical specs summary, priority
- "Database Models" — Table definitions, columns, types, constraints (if applicable)
- "API Contracts" — Endpoint specs (if applicable)

**Word document:** Create a `.docx` with professional formatting — title page, table of contents, sections for each epic/story/task, technical specification appendices with database models and API contracts.

### Stage 8 — Batch JIRA Creation

<!-- Create all tickets in JIRA with proper parent-child linking.
     Like jira-ticket-creator's Step 7 (createJiraIssue) but batched.
     
     Linking hierarchy:
       Initiative path: Initiative → Epics → Stories → Tasks
       Single Epic path: Epic → Stories → Tasks
     
     Each task description includes full technical specs so
     Mode 3 (execute) has everything it needs. -->

After outputs are generated, use `AskUserQuestion`:

- **Yes, create all tickets in JIRA** — Batch create everything now
- **No, just keep the documents** — Skip JIRA creation for now

If creating tickets:

**If Initiative-level:**

1. **Create the Initiative** using `createJiraIssue` with project key `ACD`, issue type `Epic` (tagged as Initiative in description — adapt based on available issue types).
2. **Create each Epic** using `createJiraIssue`. Link each epic to the initiative.
3. **Create each Story** using `createJiraIssue`. Link each story to its parent epic.
4. **Create each Task** using `createJiraIssue`. Include full technical specs in description. Link each task to its parent story.

**If Single Epic:**

1. **Create the Epic** using `createJiraIssue` with project key `ACD`, issue type `Epic`.
2. **Create each Story** using `createJiraIssue`. Link each story to the epic.
3. **Create each Task** using `createJiraIssue`. Include full technical specs in description. Link each task to its parent story.

**Present the results as a tree:**

```
JIRA Tickets Created:

[Initiative: INIT-KEY — title (if applicable)]
  Epic: [EPIC-KEY] — [title]
    ├── Story: [STORY-KEY] — [title]
    │   ├── [TASK-KEY] — [title]
    │   └── [TASK-KEY] — [title]
    ├── Story: [STORY-KEY] — [title]
    │   ├── [TASK-KEY] — [title]
    │   └── [TASK-KEY] — [title]
  Epic: [EPIC-KEY] — [title] (if Initiative)
    ├── ...
```

After creation, add a comment to the top-level ticket summarizing the full plan and linking to all child tickets.

**Mode 1 ends here.** No code is written. The tickets and documents are ready for future execution via `execute epic <EPIC-KEY>`.

---

<!-- ============================================================
     MODE 2 — ORCHESTRATE EPIC
     ============================================================
     Trigger: `orchestrate epic`
     
     This is Mode 1 (Plan) + Mode 3 (Execute) combined in one session.
     Runs all 8 planning stages, then immediately flows into execution.
     No need to come back later with a separate command.
     ============================================================ -->

## Mode 2 — Orchestrate Epic

**Trigger:** `orchestrate epic`

This combines planning and execution in one continuous session. It runs the full Mode 1 flow (Stages 1–8), and after JIRA tickets are created, immediately flows into Mode 3 execution — no need to come back later with a separate command.

### How It Works

1. **Run all of Mode 1** (Stages 1–8) exactly as described above — feature discovery, scope assessment, epic definitions, story breakdown, task decomposition with technical deep-dives, full plan review, output generation, and batch JIRA creation.

2. **After JIRA tickets are created**, instead of ending, ask using `AskUserQuestion`:
   - **Yes, start executing now** — Begin implementing the first epic/story
   - **No, stop here** — End the session (tickets are created, execute later)

3. **If continuing**, flow directly into Mode 3 execution (Stages 1–5) using the epic key(s) just created. The spec file is already saved, technical details are fresh in context.

---

<!-- ============================================================
     MODE 3 — EXECUTE EPIC
     ============================================================
     Trigger: `execute epic <EPIC-KEY>`
     
     Guided implementation of an existing epic.
     Only works on PENDING / unfinished tickets.
     Skips any tickets already marked Done.
     
     Works like jira-executor but at epic scale:
       - jira-executor implements ONE ticket at a time
       - execute epic implements an ENTIRE epic, story by story,
         task by task, with the same discipline
       - Both fetch the ticket, confirm understanding, present
         implementation plan, implement, then update JIRA
     
     5 stages:
       1. Load Epic Context    → Fetch from JIRA + load spec file
       2. Story-by-Story Exec  → Implement task by task with confirmation
       3. Git Commit            → Commit per story (push/no-push choice)
       4. Next Story            → Continue or pause
       5. Epic Completion       → Transition to Done + final summary
     ============================================================ -->

## Mode 3 — Execute Epic

**Trigger:** `execute epic <EPIC-KEY>` (e.g., `execute epic ACD-5`)

Implement an existing epic — only pending/unfinished tickets. Works like `jira-executor` but at epic scale: instead of implementing one ticket, it walks through the entire epic story by story, task by task, with the same discipline (confirm understanding → present approach → implement → update JIRA).

### Stage 1 — Load Epic Context

<!-- Fetch the epic and all children from JIRA.
     Load the matching spec file if one exists — it contains
     the technical details gathered during Mode 1 planning.
     Filter out tickets already marked Done. -->

Fetch the epic using `getJiraIssue`. Then fetch all linked child stories and their tasks. Also check if a matching spec file exists in `.spec/epics/` or `.spec/initiatives/` — if it does, load it as the primary reference for technical details.

**Filter to pending work only.** Skip any stories or tasks already marked as Done.

Present an overview:

```
Epic:    [EPIC-KEY] — [Epic title]
Status:  [current status]

Pending Work:
───────────────────────────────────────────────────
Story: [STORY-KEY] — [Story title] ([status])
  ☐ [TASK-KEY] — [Task title] ([status])
  ☐ [TASK-KEY] — [Task title] ([status])
  ✅ [TASK-KEY] — [Task title] (Done — skipping)

Story: [STORY-KEY] — [Story title] ([status])
  ☐ [TASK-KEY] — [Task title] ([status])
───────────────────────────────────────────────────

Pending: [X] stories, [Y] tasks
Already Done: [A] stories, [B] tasks (will be skipped)

Spec File: [path if found, or "Not found — will use ticket descriptions"]
```

Use `AskUserQuestion`:

- **Yes, let's start implementing** — Begin with the first pending story
- **No, let me pick a story** — I want to choose which story to start with

### Stage 2 — Story-by-Story Execution

<!-- For each pending story → for each pending task:
     This mirrors jira-executor's Step 1 → Step 5 flow:
       a. Fetch & present understanding (like executor Step 1)
       b. Transition to In Progress (like executor Step 2)
       c. Present implementation plan with code snippets (like executor Step 4)
       d. Get confirmation (implement / adjust / skip)
       e. Implement the code (like executor Step 5)
       f. Update JIRA with completion comment (like executor Step 7) -->

For the selected (or first pending) story, transition it to **In Progress** using `transitionJiraIssue`.

Then for each pending task in the story:

**2a. Present task understanding** (like jira-executor Step 1):

```
Task:     [TASK-KEY] — [Task title]
Status:   [current status]

Technical Specs (from plan):
[database models, API contracts, migration scripts, etc.]

My Implementation Approach:
[describe what files will be created/modified and how]
```

Use `AskUserQuestion`:

- **Yes, implement this task** — Proceed with implementation
- **No, let me adjust the approach** — I want to change something
- **Skip this task** — Move to the next task

**2b. Implement the task** (like jira-executor Step 5) — Execute the code changes as confirmed. Follow the technical specs exactly. After completion:

```
Task [TASK-KEY] Complete:

Files changed:
- [file path] — [what was done]

Acceptance Criteria:
✅ [criterion] — [how met]
```

Transition the task to **Done** in JIRA and add a completion comment (like jira-executor Step 7).

**2c. Repeat** for each pending task in the story.

After all tasks in a story are done, transition the story to **Done** and present a story summary.

### Stage 3 — Git Commit (per Story)

<!-- Like jira-executor Step 6 but separated: commit vs push.
     User might want to commit locally but not push yet. -->

After each story is complete, use `AskUserQuestion`:

- **Yes, commit and push** — Commit all changes for this story
- **Commit only, don't push** — Save locally but don't push to remote
- **No, continue without committing** — Move to the next story

If committing, use a message format like: `ACD-[STORY-KEY]: [Story summary]`

### Stage 4 — Next Story

Use `AskUserQuestion`:

- **Continue to next story** — Move to the next pending story
- **Stop here for now** — Pause execution

If stopping, add a progress comment to the Epic in JIRA summarizing what was completed and what remains.

### Stage 5 — Epic Completion

When all stories are done:

1. Transition the Epic to **Done** in JIRA.
2. Add a final comment to the Epic with a completion summary.
3. Present:

```
Epic Complete: [EPIC-KEY] — [Epic title]

Stories Implemented: [count]
Tasks Completed:     [count]
Files Changed:       [total count]

Summary of Changes:
[brief description of what was built]
```

---

<!-- ============================================================
     MODE 4 — UPDATE EPIC
     ============================================================
     Trigger: `update epic <EPIC-KEY>`
     
     Modify an existing epic's structure without executing.
     Full modification capabilities:
       - Add new stories or tasks
       - Remove stories or tasks that are no longer needed
       - Edit summaries, descriptions, acceptance criteria,
         priorities, and technical specs
       - Restructure: move tasks between stories, split/merge
         stories, reorder priorities
     
     After modifications, update JIRA tickets and spec file.
     NO code is written.
     ============================================================ -->

## Mode 4 — Update Epic

**Trigger:** `update epic <EPIC-KEY>` (e.g., `update epic ACD-5`)

Modify an existing epic's structure — add, remove, edit, or restructure stories and tasks. No code execution. Updates are reflected in both JIRA and the spec file.

### Stage 1 — Load Current State

Fetch the epic and all children using `getJiraIssue`. Load the matching spec file from `.spec/epics/` or `.spec/initiatives/` if it exists.

Present the current hierarchy:

```
Current Epic Structure: [EPIC-KEY] — [Epic title]

───────────────────────────────────────────────────
Story: [STORY-KEY] — [Story title] ([status])
  [TASK-KEY] — [Task title] ([status])
  [TASK-KEY] — [Task title] ([status])

Story: [STORY-KEY] — [Story title] ([status])
  [TASK-KEY] — [Task title] ([status])
───────────────────────────────────────────────────
```

### Stage 2 — Collect Modifications

Use `AskUserQuestion` to ask what kind of modification:

- **Add stories or tasks** — Add new work items to the epic
- **Remove stories or tasks** — Remove items that are no longer needed
- **Edit existing items** — Update summaries, descriptions, priorities, or technical specs
- **Restructure** — Move tasks between stories, split or merge stories, reorder priorities

**If Adding:**
- Ask which story to add to (or create a new story)
- Follow the same interactive flow as Mode 1 Stage 4 (for stories) or Stage 5 (for tasks with technical deep-dive)
- Confirm each addition

**If Removing:**
- Present the items and ask which to remove
- Use `AskUserQuestion` to confirm removal:
  - **Yes, remove it** — Delete from plan
  - **No, keep it** — Cancel removal
- Warn if removing a story that has in-progress tasks

**If Editing:**
- Ask which item to edit (story or task, by key)
- Show current values
- Ask for new values one field at a time
- Confirm changes

**If Restructuring:**
- Ask what to restructure (e.g., "move TASK-KEY from STORY-1 to STORY-2", "split STORY-1 into two stories", "merge STORY-2 and STORY-3")
- Present the proposed new structure
- Confirm before applying

### Stage 3 — Review Modified Plan

Present the updated hierarchy (same format as Mode 1 Stage 6) with changes highlighted:

```
Modified Epic Structure: [EPIC-KEY] — [Epic title]

Changes:
  ➕ Added: [STORY/TASK-KEY] — [title]
  ✏️  Edited: [STORY/TASK-KEY] — [what changed]
  ➖ Removed: [STORY/TASK-KEY] — [title]
  🔀 Moved: [TASK-KEY] from [STORY-A] to [STORY-B]

[Full updated hierarchy view]
```

Use `AskUserQuestion`:

- **Yes, apply these changes** — Update JIRA and spec file
- **No, make more changes** — Go back to Stage 2
- **Cancel all changes** — Discard everything

### Stage 4 — Apply Changes to JIRA and Spec File

1. **Create new tickets** for any added stories/tasks using `createJiraIssue` with proper linking.
2. **Update existing tickets** using `editJiraIssue` for edited items.
3. **Update the spec file** in `.spec/epics/` or `.spec/initiatives/` to reflect all changes.
4. **Add a comment** to the Epic ticket summarizing what was modified.

Present a summary of all changes applied.

**Mode 4 ends here.** No code is written.

---

<!-- ============================================================
     MODE 5 — UPDATE AND EXECUTE EPIC
     ============================================================
     Trigger: `update and execute epic <EPIC-KEY>`
     
     This is Mode 4 (Update) + Mode 3 (Execute) combined.
     First modify the epic structure, then execute pending tickets.
     ============================================================ -->

## Mode 5 — Update and Execute Epic

**Trigger:** `update and execute epic <EPIC-KEY>` (e.g., `update and execute epic ACD-5`)

This combines modification and execution in one session. It runs Mode 4 first (update the epic structure), then flows into Mode 3 (execute pending tickets).

### How It Works

1. **Run all of Mode 4** (Stages 1–4) — load current state, collect modifications, review, and apply changes to JIRA and spec file.

2. **After changes are applied**, ask using `AskUserQuestion`:
   - **Yes, start executing now** — Begin implementing pending tickets
   - **No, stop here** — End the session (changes are saved)

3. **If continuing**, flow directly into Mode 3 execution (Stages 1–5) with the updated epic. The spec file is freshly updated, modifications are already in JIRA.

---

<!-- ============================================================
     IMPORTANT PRINCIPLES
     ============================================================
     These rules govern behavior across ALL 5 modes. They align with
     the project's three-pillar philosophy defined in root CLAUDE.md:
       - JIRA = What to build (humans decide)
       - Spec Library = How it works (humans define)
       - Claude Code = Disciplined executor (never originates substance)
     ============================================================ -->

## Important Principles

- **Never originate substance.** All decisions about what to build, business logic, and architecture come from the user. You propose, the user decides.
- **Scale the hierarchy to fit the scope.** Large features get Initiatives with multiple epics. Focused features get a single epic. Never force unnecessary structure — but never understructure a complex feature either.
- **One question at a time.** Never combine multiple open-ended questions. Keep the interaction focused.
- **Always confirm before acting.** No JIRA tickets are created, no code is written, and no transitions happen without explicit user approval.
- **Technical specs are first-class.** Database models, migration scripts, API contracts, and component designs are part of the plan — not afterthoughts. Gather them during planning so execution is smooth.
- **The spec file is the source of truth.** In execution modes, the `.spec/` markdown file (if it exists) takes priority over JIRA ticket descriptions for technical details.
- **Epics are independently deliverable.** When breaking an Initiative into epics, each epic should be a coherent unit of work that could ship on its own.
- **Only touch pending work.** In execution modes (3, 5), skip any tickets already marked Done. Never re-execute completed work.
- **How this relates to other skills:** `jira-ticket-creator` creates single tickets. `jira-executor` implements single tickets. This orchestrator does both at epic scale — same discipline, bigger scope.
