---
name: jira-executor
description: >
  Use this skill to implement a single JIRA Task or Subtask with a disciplined,
  step-by-step workflow. Trigger on "start jira-executor", "execute ticket",
  "work on ticket", "implement ticket", "pick up ticket", or whenever the router
  dispatches a Task or Subtask. Bugs are handled by jira-bug-executor; multiple
  tickets in a row go to jira-batch-executor; a Story with its subtasks goes to
  jira-story-executor. Avoid scanning unrelated project files until Step 3 asks
  for it, so the implementation stays focused on the ticket at hand.
---

# JIRA Executor

Guide the user through implementing a JIRA ticket step by step. Every step requires explicit human confirmation before proceeding. Never skip steps or combine them — the only exception is the **no-ticket branch** described below, which explicitly skips the JIRA-only steps.

**Shared Protocols:** This skill references reusable protocols in `.claude/skills/jira/_shared/references/`. Read the relevant protocol file when a step says "follow [protocol-name]".

- `human-confirmation-protocol.md` — Enhanced confirmation patterns (all steps) **including `hcp.8.remember-answer` — see below**
- `spec-first-protocol.md` — Verify .spec coverage before implementation (Step 3)
- `git-workflow.md` — Git operations (Step 8, Level 1)

**Dev Flows Rules:** Step 2 produces per-feature dev-flow artefacts under `.spec/dev-flows/feature/<feature-name>/` (default) or as JIRA comments (if the user chooses), using the rules in `.claude/jira/jira-executor-rules.md` (and its sub-rules `code-to-phrase-generator.md`, `phrase-to-sentence-generator.md`, `spec-id-conventions.md`).

## Per-skill rule — Remember answers for the session {#exc.remember}

**Every `AskUserQuestion` raised by this skill MUST follow `hcp.8.remember-answer` from `human-confirmation-protocol.md`.** Concretely:

- For storage / format / routing questions (Steps 2 Prompts 1 & 2, Step 4 "guide me" choices, Step 8 commit-and-push choices, etc.), use **Pattern A**: pair each main option with a sibling `Remember: always <X> this session` option.
- For free-form confirmations (Steps 1, 5, 6 confirmations where the right answer depends on the ticket's unique content), use **Pattern B**: append a single `Remember my answer for this session` toggle, honoured only for questions with an identical prompt + option set.
- A remembered answer is **session-scoped only** — never persisted to memory files, never carried across sessions.
- **NEVER offer a remember option for destructive/irreversible actions** (Step 7 transition, Step 8 push, Step 9 JIRA comment post, any spec-file write). Those are covered by `hcp.6.destructive-pattern` and must be asked fresh every time.
- When a remembered answer is applied, print one line: `▸ Using remembered choice: "<answer>"`.

## Section IDs

**Prefix:** `exc`

IDs follow the [Spec ID Conventions](../../../jira/jira-executor-rules/spec-id-conventions.md) — all lowercase, kebab-case segments, dots between, and each step uses a sequential `step-N` form (no sub-letters).

When starting each step, display the section ID:

```
▶ [exc.step-N.name] — Step title
```

| ID | Step |
|---|---|
| `exc.step-1.fetch-understand` | Fetch and present ticket understanding (ticket path) |
| `exc.step-2.dev-flows` | Generate dev flows (code-to-phrase + code-to-sentence), pick storage, verify `.spec/dev/` |
| `exc.step-3.spec-first-check` | Spec-First Check |
| `exc.step-4.identify-specs` | Identify relevant spec files |
| `exc.step-5.implementation-plan` | Present implementation plan (strict 1:1 with code-to-phrase) |
| `exc.step-6.implement-code` | Implement the code (strict 1:1 with code-to-phrase) |
| `exc.step-7.transition-progress` | Transition ticket to **In Progress** (after the code is complete and ready) |
| `exc.step-8.git-workflow` | Git workflow (separate commits) |
| `exc.step-9.jira-update` | Final feedback and JIRA update |

## Flow

### No-ticket branch

If the user declined to create a JIRA ticket for this work (e.g., when the ticket-creation prompt driven by root `CLAUDE.md` was answered **No**), do **not** abort. Run the flow in **no-ticket mode**:

1. **Replace Step 1** with a free-form requirements gather. Use `AskUserQuestion` (and follow-up prompts) to collect the feature summary, goal, acceptance criteria, priority, and any constraints. Iterate until you have enough to restate the understanding back to the user and get explicit confirmation — the confirmation pattern mirrors Step 1. In this branch, Prompt 1 (code-to-phrase storage) and Prompt 2 (code-to-sentence storage) **only allow "Spec file" and "Skip"** — JIRA comment is not available because there is no ticket to comment on.
2. **Skip every JIRA-only step**: **Step 7 (transition-progress)** and **Step 9 (JIRA update comment)** do not run. Any `getJiraIssue` / `transitionJiraIssue` / `addCommentToJiraIssue` calls described in those steps are omitted.
3. **Run all remaining exc steps normally** — Step 2 (Dev Flows), Step 3 (Spec-First Check), Step 4 (Identify Specs), Step 5 (Implementation Plan), Step 6 (Implement Code), Step 8 (Git Workflow) — using the user-supplied details in place of ticket fields.

Otherwise, proceed with the ticket path starting at Step 1.

### Step 1 — Fetch and present ticket understanding {#exc.step-1.fetch-understand}

Fetch the JIRA ticket using the `getJiraIssue` MCP tool. Read the ticket summary, description, and acceptance criteria thoroughly.

**Bug-handoff fast path.** If the ticket description contains a block beginning with `## Analysis (added by jira-bug-executor on ` — i.e. `jira-bug-executor` has already walked the user through the diagnosis and written it to the description — treat the analysis as already confirmed. Show a brief note:

```
▸ Analysis block detected on [KEY] (added by jira-bug-executor).
  Treating understanding as already confirmed — skipping the Step 1 re-confirm.
  Proceeding to Step 2.
```

Do NOT re-ask the understanding confirmation in this case. Move directly to Step 2.

Otherwise (no analysis block), present your full understanding back to the user:

```
Ticket:      [KEY] — [Summary]
Type:        [Issue Type]
Priority:    [Priority]
Status:      [Current Status]

My Understanding:
[Rephrase the ticket requirements in your own words — what needs to be done, why, and what the expected outcome is]

Acceptance Criteria:
- [criterion 1]
- [criterion 2]
- ...
```

Then use `AskUserQuestion` to confirm (per `hcp.2.understanding-pattern`, with a Pattern B `Remember my answer` toggle — `hcp.8`):

- **Yes, that's correct** — Your understanding is accurate, proceed
- **Partially correct, let me clarify** — Some parts need correction
- **I want to add more context** — There are additional details you should know
- **Remember my answer for this session** — Apply the same confirmation pattern to every identical understanding prompt this session (use sparingly; understandings usually differ per ticket)

If the user selects **"Partially correct"** or **"add more context"**, collect their input, update your understanding, and present it again. Repeat until confirmed.

### Step 2 — Dev Flows Generation {#exc.step-2.dev-flows}

**Follow the rules in `.claude/jira/jira-executor-rules.md`** (and its sub-rules: `code-to-phrase-generator.md`, `phrase-to-sentence-generator.md`, `spec-id-conventions.md`).

This step is the authoritative checklist phase — the output of Step 2 becomes the strict scope for Steps 5 and 6.

#### 2.1 — Derive the `<feature-name>`

Derive a kebab-case `<feature-name>` from the ticket summary (and, if needed, the ticket key). Propose it to the user and confirm:

```
Proposed feature name: <feature-name>
  Rationale: [why this name was chosen, e.g., "matches the ticket summary 'Forgot password reset' and existing .spec/dev/features/ naming convention"]
```

Use `AskUserQuestion` (Pattern A per `hcp.8`):

- **Yes, use `<feature-name>`** — Accept the proposed name
- **Let me pick a different name** — Provide a corrected kebab-case name
- **Remember my answer for this session** — Optional; rarely useful since the name changes per feature

Follow the project [naming conventions](../../../../.spec/dev/rules/naming-rules.md) for kebab-case.

#### 2.2 — Produce `code-to-phrase`

Draft the ordered phrase list for every planned code change for this ticket, authored strictly per the [Code to Phrase Generator](../../../jira/jira-executor-rules/code-to-phrase-generator.md) rules. Do NOT persist it yet — present it inline first.

**Prompt 1 — Where should `code-to-phrase` be stored?** (Pattern A, `hcp.8` — honour any remembered session choice before asking.)

- **Spec file (default)** — Save to `.spec/dev-flows/feature/<feature-name>/code-to-phrase.md`
- **JIRA comment** — Post the code-to-phrase as a comment on this ticket (via `addCommentToJiraIssue`, with heading `### code-to-phrase`)
- **Remember: always Spec file this session** — Don't ask again; default to Spec file for every subsequent ticket this session
- **Remember: always JIRA comment this session** — Don't ask again; default to JIRA comment for every subsequent ticket this session
- **Skip storing for this ticket** — Don't persist the code-to-phrase (only if the user explicitly says it isn't worth keeping)

Apply the user's (or remembered) choice. Persist the artefact. Show `▸ Stored code-to-phrase at <location>`.

In no-ticket mode, the "JIRA comment" and its remember variant are **not available** — show only Spec file / Skip.

#### 2.3 — Confirm the phrase list

Show the code-to-phrase back to the user, then use `AskUserQuestion`:

- **Phrases look right, proceed** — Lock this as the strict scope; advance to 2.4
- **I want to edit the phrases** — Revise inline and re-present
- **Show me the phrases again** — Re-display without editing

Once the user confirms, the phrase list is **locked** as the authoritative scope. From this point, Steps 5 and 6 cannot exceed it without re-opening Step 2 (see the strict-scope rule below).

#### 2.4 — Produce `code-to-sentence`

**Prompt 2 — Generate `code-to-sentence`, and where should it be stored?** (Pattern A, `hcp.8` — honour remembered session choice before asking.)

- **Generate and store in Spec file (default)** — Expand each phrase into `.spec/dev-flows/feature/<feature-name>/code-to-sentence.md`
- **Generate and store in JIRA comment** — Post expanded paragraphs as a comment (heading `### code-to-sentence`)
- **Skip code-to-sentence for this ticket** — Do not generate at all
- **Remember: always Spec file this session** — Always generate, always store in Spec file
- **Remember: always JIRA comment this session** — Always generate, always store in JIRA comment
- **Remember: always skip this session** — Never generate code-to-sentence this session

In no-ticket mode, the "JIRA comment" option and its remember variant are not available.

If the user picks generate-and-store, expand each phrase 1:1 into a descriptive paragraph per the [Phrase to Sentence Generator](../../../jira/jira-executor-rules/phrase-to-sentence-generator.md) rules, using the same section headers, same order, and same phrase wording as the confirmed code-to-phrase. Persist the artefact. Show `▸ Stored code-to-sentence at <location>`.

If the user picks Skip, note it — `code-to-sentence` is omitted for this ticket.

#### 2.5 — Verify `.spec/dev/` coverage

Scan every spec file and section already under `.spec/dev/` (rules, architecture, epics, features) and list what is already in place that supports the dev flows just authored. **Do not scan `.spec/dev-flows/`** — that directory only contains the phrase/sentence artefacts generated above; coverage verification runs against `.spec/dev/` only.

Present the summary:

```
Dev Flows Generated:
- code-to-phrase   → [Spec file: .spec/dev-flows/feature/<feature-name>/code-to-phrase.md | JIRA comment on [KEY] | Skipped]
- code-to-sentence → [Spec file: .spec/dev-flows/feature/<feature-name>/code-to-sentence.md | JIRA comment on [KEY] | Skipped]

.spec/dev coverage for the current dev flows:
- [file path] — [section ids found, relevance note]
- [file path] — [section ids found, relevance note]
(or "No existing .spec/dev coverage found — all supporting specs will need to be created in Step 3.")
```

### Step 3 — Spec-First Check {#exc.step-3.spec-first-check}

**Follow the Spec-First Protocol** in `.claude/skills/jira/_shared/references/spec-first-protocol.md`.

Before writing any code, verify that all `.spec` files and sections required for this ticket exist. If any are missing:

1. Present what's missing to the user
2. Interactively gather the spec details
3. Create the `.spec` file/section (under `.spec/dev/...`)
4. Register it in root `CLAUDE.md` under Registered Specs (`cmd.7.registered-specs`)
5. Confirm with the user before proceeding

Only after all required specs are in place, move to the next step.

### Step 4 — Identify relevant spec files {#exc.step-4.identify-specs}

Scan all `.spec/dev/` markdown files in the project. For each file that is relevant to implementing this ticket, list it with its sections and IDs:

```
Relevant Spec Files:

1. [file path]
   - [section-id] — [section title/description]
   - [section-id] — [section title/description]

2. [file path]
   - [section-id] — [section title/description]
```

If no spec files are relevant, state that clearly.

Then use `AskUserQuestion` to confirm (Pattern B, `hcp.8`):

- **Yes, this is correct** — These are the right spec files and sections
- **No, let me adjust** — I want to add or remove sections
- **Guide me** — Show me all available spec files so I can pick
- **Remember my answer for this session** — Optional

If the user selects **"No, let me adjust"**, ask which sections to add or remove, then present the updated list. Repeat until confirmed.

If the user selects **"Guide me"**, list ALL available `.spec/dev/` markdown files with ALL their sections and IDs. Let the user pick which ones are relevant. Then present the final selection for confirmation.

### Step 5 — Present implementation plan (strict 1:1 with code-to-phrase) {#exc.step-5.implementation-plan}

**Hard rule — the plan is a 1:1 projection of the confirmed `code-to-phrase`.** Every plan item MUST map to exactly one phrase, and every phrase MUST map to at least one plan item. No extras.

Build the plan by walking the phrase list in order. For each phrase:

- Name the file path(s) it touches.
- Name the action (`create` / `modify` / `delete`).
- Show only the minimal code snippet required by that phrase.
- Tag the plan item with the phrase it traces to.

Present the plan in the exact format below — the trailing `(phrase: "…")` tag is **required** so the traceability is visible:

```
Implementation Plan (1:1 with code-to-phrase):

1. [File path — new/modified]
   Action:  [create / modify / delete]
   Impact:  [what this change does]
   (phrase: "[exact phrase from code-to-phrase]")

   Code snippet:
   [the minimal code the phrase requires — not the full file]

2. [File path — new/modified]
   Action:  ...
   (phrase: "[exact phrase from code-to-phrase]")
   ...
```

**Forbidden in the plan (enforcement per `jira-exec.strict-scope`):** helper stubs, `.gitkeep`, `.gitignore`, `README.md`, unsolicited tests, nearby refactors, extra imports/exports/configs/env-vars/feature-flags/CI tweaks, "while we're here" tidy-ups — **unless** a phrase explicitly names them. If during planning you realise a genuinely required change is missing from the phrase list, STOP, return to Step 2.3, explain the missing phrase, get explicit user confirmation, add it to `code-to-phrase` (and if applicable to `code-to-sentence`), then resume the plan.

If you believe something additional is beneficial but no phrase covers it, list it **separately** as a "Suggestion (not in scope unless you add it)" block — it MUST NOT appear in the numbered plan until the phrase list is extended.

Then use `AskUserQuestion` (Pattern B, `hcp.8`):

- **Yes, proceed** — Looks good, start implementing
- **No, let me adjust** — I want to change the approach (return to Step 2.3 if the change needs new phrases)
- **Looks good but I want to add more details** — Additional constraints before proceeding (return to Step 2.3 if the detail introduces new phrases)
- **Remember my answer for this session** — Optional

If the user adjusts or adds details, revise the plan (and the phrase list if needed) and present again. Repeat until confirmed.

### Step 6 — Implement the code (strict 1:1 with code-to-phrase) {#exc.step-6.implement-code}

Only after the user confirms the implementation plan, execute the actual code changes. Follow the plan exactly as confirmed.

**Strict-scope discipline (per `jira-exec.strict-scope`):**

- Implement **only** what the confirmed phrase list and plan describe. Do not add files, helpers, tests, refactors, configs, or glue that isn't named.
- If during coding you discover a genuinely required change that has no matching phrase (e.g., a missing DB index, a mandatory config entry, a compile-time dependency), STOP. Do NOT write the extra code. Return to Step 2.3, surface the discovery to the user, get explicit confirmation, add the phrase to `code-to-phrase` (and expand it into `code-to-sentence` if sentences are being stored) **before** writing the corresponding code.
- Never bundle unrelated changes into one file edit "while I'm in there".

After implementation is complete, run the **post-implementation reconciliation**:

1. For every changed file, confirm it traces to a phrase.
2. For every phrase, confirm at least one change was made.
3. If either check fails, surface the discrepancy in the summary below and resolve before moving on.

Present a summary:

```
Implementation Complete:

Files changed:
- [file path] — [what was done]     (phrase: "[exact phrase]")
- [file path] — [what was done]     (phrase: "[exact phrase]")

Phrase coverage:
✅ [phrase 1] — implemented in [file(s)]
✅ [phrase 2] — implemented in [file(s)]
(If any phrase is ❌ unimplemented or any file has no phrase, flag it here.)

Acceptance Criteria Check:
✅ [criterion 1] — [how it was met]
✅ [criterion 2] — [how it was met]
```

Then use `AskUserQuestion` (Pattern B, `hcp.8`):

- **Yes, looks good** — I'm satisfied with the changes
- **No, something needs fixing** — I see an issue that needs correction
- **Run it again with modifications** — I want to adjust and re-implement
- **Remember my answer for this session** — Optional

If the user selects **"No, something needs fixing"** or **"Run it again with modifications"**, loop back, fix/adjust, and re-present the summary. Only when the user selects **"Yes, looks good"** does the flow advance to Step 7.

### Step 7 — Transition ticket to In Progress {#exc.step-7.transition-progress}

**Skip this step entirely in no-ticket mode.**

Run this step only after Step 6 has been confirmed "Yes, looks good" — i.e., the code is complete and ready. Transition the JIRA ticket to **In Progress** using the `transitionJiraIssue` MCP tool.

The transition is deliberately late: the ticket is flipped to *In Progress* only once the implementation is actually in a ready state, so the status reflects real working state when a reviewer picks it up. Do **not** transition earlier — leaving the ticket in its prior status until the code is ready keeps JIRA aligned with reality.

This is a destructive/irreversible action per `hcp.6.destructive-pattern` — **do NOT offer a remember option**. Confirm once, per ticket, before the transition:

- **Yes, transition to In Progress** — Flip the status
- **Wait, let me review first** — Pause the flow
- **No, skip the transition** — Leave the status as-is (rare; note it for the audit trail)

After the transition, confirm the new status back to the user before moving on to Step 8.

### Step 8 — Git workflow {#exc.step-8.git-workflow}

**Follow the Git Workflow Protocol (Level 1 — Basic)** in `.claude/skills/jira/_shared/references/git-workflow.md`.

<!-- ============================================================
     SEPARATE COMMIT PROTOCOL
     ============================================================
     When a ticket involves changes to multiple categories of files
     (spec files, dev-flow files, implementation code), commit each
     category separately. This keeps the git history clean and
     makes it easy to review changes by concern.

     Commit order:
       1. Spec files (.spec/dev/)           — if created/updated
       2. Dev flow files (.spec/dev-flows/) — if created/updated (only if dev-flows were stored as spec files)
       3. Implementation code               — the actual code changes
     ============================================================ -->

Before committing, categorize all changed files:

```
Changed Files Summary:

Spec files (.spec/dev/, .spec/rules/, etc. — excluding .spec/dev-flows/):
- [file] — [created/updated]

Dev flow files (.spec/dev-flows/):
- [file] — [created/updated]  (only present if dev-flows were stored as Spec file; absent if stored as JIRA comments or skipped)

Implementation code:
- [file] — [created/modified]
```

**Commit each category separately, in this order:**

1. **Spec files first** (if any were created/updated):
   - Commit message: `[KEY]: Add/Update spec — [brief description]`

2. **Dev flow files second** (if any were created/updated):
   - Commit message: `[KEY]: Add/Update dev flows for [feature-name]`

3. **Implementation code last:**
   - Commit message: `[KEY]: [Implementation summary]`

For each commit, use `AskUserQuestion` (Pattern A, `hcp.8` — commit decisions are reusable per session):

- **Commit and push** — Commit and push the changes to remote
- **Commit only, don't push** — Save locally but don't push
- **Skip git for this category** — Leave changes uncommitted for now
- **Remember: always commit and push this session** — Don't re-ask; commit-and-push every category this session
- **Remember: always commit only this session** — Don't re-ask; commit (no push) every category this session

Pushing is destructive per `hcp.6` — a remembered "commit and push" still prompts fresh once before the FIRST push of the session, then auto-applies after that ONLY if the user reconfirmed. Do not remember the push decision silently.

If only one category has changes, just do a single commit as before.

In **no-ticket mode**, use a short descriptive prefix (e.g., `[no-ticket]` or the feature name) instead of `[KEY]` in commit messages.

### Step 9 — Final feedback and JIRA update {#exc.step-9.jira-update}

Branch by mode:

#### Ticket mode (default)

Produce the final summary, then post it to the JIRA ticket.

```
Final Summary:

Ticket:    [KEY] — [Summary]
Status:    In Progress (transitioned in Step 7)
Changes:   [brief list of what was implemented]
Specs:     [created/updated/none]
Dev flows: [stored at: Spec file <path> | JIRA comment on [KEY] | Skipped — for each of code-to-phrase / code-to-sentence]
Git:       [N commits — committed/pushed/none]
Notes:     [any observations, risks, or follow-up items]
```

Then update the JIRA ticket:

1. **Add a comment** to the ticket with the final summary and any major observations using the `addCommentToJiraIssue` MCP tool.
2. If there are any major updates, blockers, or risks discovered during implementation, add those as a separate comment.
3. Do **NOT** transition the ticket to "Done" — leave it in "In Progress" (transitioned in Step 7) for human review. Only the human decides when a ticket is Done.

Posting a JIRA comment is destructive per `hcp.6` — do NOT offer a remember option for "should I post?".

Use `AskUserQuestion` for final wrap-up (Pattern B, `hcp.8`):

- **Done, that's all** — End the session
- **I want to add a note to the ticket** — I have additional observations to record
- **Start another ticket** — I want to execute another ticket next
- **Remember my answer for this session** — Optional

#### No-ticket mode

Produce just the Final Summary block for the user and end. Do NOT call `addCommentToJiraIssue`, `transitionJiraIssue`, or any JIRA-write tool.

```
Final Summary (no-ticket):

Work:      [short label derived from the user-supplied summary]
Changes:   [brief list of what was implemented]
Specs:     [created/updated/none]
Dev flows: [Spec file <path> | Skipped — for each of code-to-phrase / code-to-sentence]
Git:       [N commits — committed/pushed/none]
Notes:     [any observations, risks, or follow-up items]
```
