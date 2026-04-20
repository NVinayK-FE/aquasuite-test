# Spec-First Protocol

<!-- ============================================================
     SHARED PROTOCOL: Spec-First Execution
     ============================================================
     Used by: jira-executor only (invoked from its Step 3).
              Upstream orchestrators (jira-batch-executor,
              jira-story-executor, jira-epic-orchestrator,
              jira-bug-executor) never run this protocol directly —
              they delegate per-ticket execution to jira-executor,
              which in turn runs this protocol once per ticket.

     Scope: audits only the current repository's .spec/ tree. Cross-
     project batches need one spec-first pass per repo — see
     jira-batch-executor's "Cross-project spec caveat" for warnings.

     Ordering note: as of the current jira-executor flow, this
     protocol runs AFTER the dev-flows generation step (code-to-
     phrase / code-to-sentence produced in jira-executor Step 2).
     The phrase list is the authoritative scope of the change —
     this protocol verifies that the specs needed to support those
     phrases exist and are complete.

     Purpose: Before implementing ANY ticket/task, perform a deep
     spec audit at BOTH file AND section level. Identify every
     missing spec file, missing section, and incomplete section
     that the task requires. Create or update ALL specs FIRST,
     register in CLAUDE.md, then — and only then — implement.
     
     This ensures the spec library is always complete before
     code is written — the spec is the contract, code follows it.
     
     Philosophy: Specs are not just documentation — they are the
     single source of truth for HOW the system works. Every
     implementation decision must trace back to a spec. If a spec
     doesn't exist, the decision hasn't been made yet, and the
     human must make it before code is written.
     ============================================================ -->

## Section IDs

**Prefix:** `spec`

| ID | Section |
|---|---|
| `spec.1.identify-required` | Identify Required Specs (Deep Analysis) |
| `spec.2.scan-existing` | Scan Existing Specs (File + Section + ID Level) |
| `spec.3.gap-report` | Gap Report & Human Input |
| `spec.4.create-update-specs` | Create / Update Specs |
| `spec.5.final-confirmation` | Final Spec Confirmation |

## When to Run

Run this protocol **before every implementation step** — whether executing a single ticket (jira-executor), a batch of tickets (jira-batch-executor), or tasks within an epic (jira-epic-orchestrator Mode 3/5).

**No code is written until all required specs are confirmed.** This is a hard gate.

---

## Protocol Steps

### Step 1 — Identify Required Specs (Deep Analysis) {#spec.1.identify-required}

Analyze the ticket/task requirements and determine **every** `.spec` file and section needed for implementation. Think broadly — not just what the ticket explicitly mentions, but what a developer would need to implement it correctly and consistently.

**Categories to check (evaluate ALL of these for every task):**

| Category | When to check | Example spec IDs |
|---|---|---|
| **Naming conventions — folders** | Task creates new folders | `naming-rule.folder` |
| **Naming conventions — files** | Task creates new files | `naming-rule.file` |
| **Naming conventions — components** | Task creates React/UI components | `naming-rule.component` |
| **Naming conventions — variables/functions** | Task writes code with exports | `naming-rule.variable`, `naming-rule.function` |
| **Architecture — project structure** | Task sets up or modifies project structure | `architecture.project-structure` |
| **Architecture — module pattern** | Task creates a new module or pattern | `architecture.module-pattern` |
| **Architecture — provider pattern** | Task involves context providers | `architecture.provider-pattern` |
| **Architecture — state management** | Task involves stores/state | `architecture.state-management` |
| **Database schemas** | Task involves tables, models, or migrations | `db-schema.<entity>` |
| **API contracts** | Task involves endpoints or API calls | `api-contract.<resource>` |
| **Business rules** | Task involves validation, logic, or policies | `rule.<domain>.<rule-name>` |
| **Config specs** | Task involves environment, build, or tool config | `config.<tool-name>` |
| **Epic/feature specs** | Task belongs to an epic with a spec file | `<epic-id>` |

**Output a requirements list** — every spec file and section the task needs, with reasoning:

```
Spec Requirements Analysis for [KEY]:

This task requires the following specs:

1. naming-rule.folder    — Task creates new directories (src/app/, src/components/, etc.)
2. naming-rule.file      — Task creates new files (layout.tsx, page.tsx, etc.)
3. architecture.project-structure — Task defines the project's folder hierarchy
4. config.typescript      — Task configures tsconfig.json with specific settings
5. nextjs-web-frontend-foundation — Parent epic spec with technical architecture

Reasoning: [Brief explanation of why each is needed]
```

---

### Step 2 — Scan Existing Specs (File + Section + ID Level) {#spec.2.scan-existing}

Perform a **deep scan** of all `.spec/` markdown files. This is NOT a surface-level file check — read each relevant spec file and inspect it at the **section level using section IDs**.

**Scan procedure:**

1. **Read the Registered Specs table** in root `CLAUDE.md` — get all registered spec file IDs and paths.

2. **For each required spec from Step 1**, check:
   - Does the **spec file** exist on disk?
   - Is the file **registered** in CLAUDE.md?
   - Does the required **section** (identified by `<!-- id: X.Y -->` comment) exist within the file?
   - Is the section **complete enough** for this task, or does it need additional content?

3. **Read the actual spec files** — do not just check file existence. Open each relevant file and scan its sections. Look for `<!-- id: ... -->` markers to identify sections.

**Present findings in a detailed table:**

```
▶ [spec.2.scan-existing] — Deep Spec Scan Results

Registered Specs in CLAUDE.md:
  • naming-rule          → .spec/rules/naming-rules.md
  • nextjs-web-frontend-foundation → .spec/epics/nextjs-web-frontend-foundation.md

Detailed Section Audit:
═══════════════════════════════════════════════════════════════

FILE: .spec/rules/naming-rules.md (registered as: naming-rule)
───────────────────────────────────────────────────────────────
  ✅ naming-rule.folder       — Found: kebab-case folder naming rules
  ⚠️  naming-rule.file        — MISSING SECTION: No file naming conventions defined
  ⚠️  naming-rule.component   — MISSING SECTION: No component naming conventions
  ⚠️  naming-rule.variable    — MISSING SECTION: No variable/function naming conventions

FILE: .spec/epics/nextjs-web-frontend-foundation.md (registered as: nextjs-web-frontend-foundation)
───────────────────────────────────────────────────────────────
  ✅ Technical architecture   — Found: folder structure, tech stack, provider nesting
  ✅ Story 1 task details     — Found: scaffolding commands and config specs

MISSING FILES:
───────────────────────────────────────────────────────────────
  ❌ .spec/architecture/       — No architecture spec file exists
     Missing sections needed:
     • architecture.project-structure — Project folder hierarchy and conventions
     • architecture.module-pattern    — Module organization pattern

  ❌ .spec/config/              — No config spec file exists
     Missing sections needed:
     • config.typescript — TypeScript configuration standards

═══════════════════════════════════════════════════════════════

Summary:
  Total required:    [N] specs/sections
  ✅ Found & complete: [N]
  ⚠️  Found but incomplete: [N] (section missing in existing file)
  ❌ Missing entirely: [N] (file or section does not exist)
```

**Important rules for this step:**

- Always show section IDs (e.g., `naming-rule.file`) not just file names
- Distinguish between "file exists but section is missing" (⚠️) vs "file doesn't exist" (❌)
- For sections that exist, assess if they are **complete enough** for the current task — a section with only one rule may need expansion
- Flag sections that exist but may need **updates** for this task (e.g., existing folder naming rules that don't cover the specific patterns this task introduces)

---

### Step 3 — Gap Report & Human Input {#spec.3.gap-report}

If there are ANY gaps (⚠️ or ❌), present a clear gap report and ask for human input.

**Do NOT proceed to implementation.** Specs must be resolved first.

```
▶ [spec.3.gap-report] — Spec Gaps Detected

[N] spec gaps must be resolved before implementation:

MISSING SECTIONS in existing files:
  1. naming-rule.file — File naming conventions (e.g., kebab-case for files? PascalCase for components?)
  2. naming-rule.component — React component naming (e.g., PascalCase? default vs named exports?)
  3. naming-rule.variable — Variable and function naming (e.g., camelCase?)

MISSING FILES:
  4. .spec/architecture/project-structure.md — Project folder hierarchy, module boundaries
  5. .spec/config/typescript-config.md — TypeScript configuration standards

POTENTIALLY INCOMPLETE:
  6. naming-rule.folder — Exists but only covers generic kebab-case; may need Next.js-specific rules
     (e.g., route groups like (auth), (dashboard), dynamic routes like [id])

I recommend creating/updating all of these before starting implementation.
Do you have any additional specs or conventions you'd like to add?
```

Use `AskUserQuestion`:

- **Yes, let's create all missing specs now** — Walk me through defining them one by one
- **I have additional specs to add** — I want to specify more conventions or rules before we start
- **Create what you can, ask me for the rest** — Auto-generate reasonable defaults, I'll review and adjust
- **Skip specs for now, use ticket description only** — I'll define specs later (not recommended)

**If user selects "I have additional specs to add":**

Ask: "What additional specs, conventions, or rules would you like to define? (e.g., import ordering, CSS class naming, test file naming, git branch naming, etc.)"

Record the additions and add them to the gap list.

**If user selects "Create what you can, ask me for the rest":**

For each gap, generate a reasonable default based on:
- Industry best practices for the tech stack (Next.js, React, TypeScript)
- Patterns already established in existing specs
- Conventions from the epic spec file

Present ALL auto-generated specs together for review before saving.

---

### Step 4 — Create / Update Specs {#spec.4.create-update-specs}

Process each gap one by one (or in batches if the user chose auto-generation).

#### For MISSING SECTIONS in existing files:

1. Present the proposed new section with its section ID marker:

```
Adding to: .spec/rules/naming-rules.md

New section:

## File Naming <!-- id: naming-rule.file -->

> Rules for naming files across the project.

- Use lowercase `kebab-case` for all non-component files (e.g., `api-client.ts`, `use-api-query.ts`)
- Use `PascalCase` for React component files (e.g., `ThemeToggle.tsx`, `DataTable.tsx`)
- [more rules...]
```

2. Use `AskUserQuestion`:
   - **Yes, this section is correct** — Add it to the file
   - **No, let me adjust** — I want to modify the content
   - **Add more details to this section** — There's more to include

3. After confirmation, **append the section** to the existing spec file.

#### For MISSING FILES:

1. Present the proposed new file with all required sections:

```
Creating: .spec/architecture/project-structure.md

# Project Structure

## Overview <!-- id: architecture.project-structure -->

> Defines the standard project folder hierarchy and module boundaries.

[content...]
```

2. Use `AskUserQuestion` to confirm (same options as above).

3. After confirmation:
   - **Create the file** in the appropriate `.spec/` subdirectory
   - **Register in root CLAUDE.md** under Registered Specs with proper ID

```
Spec Created:
  File:     .spec/architecture/project-structure.md
  Section:  architecture.project-structure
  Registered in CLAUDE.md: ✅
```

#### For INCOMPLETE/NEEDS-UPDATE sections:

1. Show the **current content** and the **proposed additions**:

```
Updating: .spec/rules/naming-rules.md → naming-rule.folder

Current content:
  - Use lowercase kebab-case for all folder names
  - Avoid abbreviations unless widely understood
  - Group related files under descriptive folder name

Proposed additions:
  - Next.js route groups use parentheses: (auth), (dashboard)
  - Dynamic route segments use brackets: [id], [slug]
  - [more additions...]
```

2. Confirm with `AskUserQuestion` before saving.

#### Progress tracking:

After each spec is created/updated, show progress:

```
Spec Progress: [3/6] resolved

  ✅ [1] naming-rule.file — Added to .spec/rules/naming-rules.md
  ✅ [2] naming-rule.component — Added to .spec/rules/naming-rules.md
  ✅ [3] naming-rule.variable — Added to .spec/rules/naming-rules.md
  ▶  [4] architecture.project-structure — Next
  ☐  [5] config.typescript
  ☐  [6] naming-rule.folder (update)
```

---

### Step 5 — Final Spec Confirmation {#spec.5.final-confirmation}

After ALL gaps are resolved, present the complete spec coverage:

```
▶ [spec.5.final-confirmation] — All Specs Ready

═══════════════════════════════════════════════════════════════

All Required Specs for [KEY]:

  ✅ naming-rule.folder             — .spec/rules/naming-rules.md (updated)
  ✅ naming-rule.file               — .spec/rules/naming-rules.md (just created)
  ✅ naming-rule.component          — .spec/rules/naming-rules.md (just created)
  ✅ naming-rule.variable           — .spec/rules/naming-rules.md (just created)
  ✅ architecture.project-structure — .spec/architecture/project-structure.md (just created)
  ✅ config.typescript              — .spec/config/typescript-config.md (just created)
  ✅ nextjs-web-frontend-foundation — .spec/epics/nextjs-web-frontend-foundation.md (existing)

Files created:  [N]
Files updated:  [N]
Sections added: [N]

CLAUDE.md updated with [N] new registrations.

═══════════════════════════════════════════════════════════════

All specs are confirmed. Ready to proceed with implementation.
```

Use `AskUserQuestion`:

- **Yes, proceed with implementation** — All specs look good, start coding
- **No, I want to review the specs again** — Let me double-check before coding
- **I want to add one more thing** — Additional spec before we start

**Only after this confirmation does implementation begin.** This is the hard gate.

---

## Key Principles

1. **Specs before code — always.** No exceptions. Even for "simple" scaffolding tasks, the naming conventions, architecture patterns, and config standards must be defined first.

2. **Section-level granularity.** A spec file existing is not enough. The specific section with its ID must exist and be complete for the task at hand.

3. **Section IDs are mandatory.** Every section in a spec file must have an `<!-- id: X.Y -->` marker. This enables precise cross-referencing between specs and implementation tasks.

4. **Human decides, Claude executes.** Claude can propose spec content based on best practices, but the human confirms every spec before it's saved. Claude never originates substance.

5. **Future-proof.** When creating specs, think beyond the current task. If a naming rules section is being created, define conventions that will serve all future tasks — not just the immediate one.

6. **Update existing specs.** If a section exists but is incomplete for the current task's needs, flag it for update rather than skipping it.

7. **Register everything.** Every new spec file gets registered in CLAUDE.md. Every section gets an ID. No orphan specs.
