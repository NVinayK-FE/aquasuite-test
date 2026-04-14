# Acceptance Criteria Checklist Protocol

<!-- ============================================================
     SHARED PROTOCOL: Acceptance Criteria as Interactive Checklist
     ============================================================
     Used by: jira-ticket-creator, jira-epic-orchestrator (for stories and tasks)
     
     Purpose: Instead of asking the user to write acceptance criteria
     from scratch, SUGGEST criteria based on the ticket type and
     description, present them as a checklist for the user to
     select/deselect, and offer to add custom criteria.
     
     This reduces cognitive load on the user while ensuring
     comprehensive coverage.
     ============================================================ -->

## Section IDs

**Prefix:** `acc`

| ID | Section |
|---|---|
| `acc.1.generate-criteria` | Generate Suggested Criteria |
| `acc.2.present-checklist` | Present as Checklist |
| `acc.3.handle-selection` | Handle User Selection |
| `acc.4.final-confirmation` | Final Confirmation |

## When to Use

Use this protocol whenever acceptance criteria need to be defined — during ticket creation (jira-ticket-creator) and during epic planning (stories and tasks in jira-epic-orchestrator).

## Protocol Steps

### Step 1 — Generate Suggested Criteria {#acc.1.generate-criteria}

Based on the ticket type, summary, and description already collected, generate a set of suggested acceptance criteria. These should be specific, testable, and relevant to the work described.

**For database tasks**, suggest criteria like:
- Tables created with correct columns and types
- Foreign keys and indexes defined
- Migration script runs without errors
- Rollback script available and tested
- Seed data loads correctly

**For API tasks**, suggest criteria like:
- Endpoint returns correct status codes
- Request validation rejects invalid payloads
- Response matches documented schema
- Authentication/authorization enforced
- Error responses follow standard format

**For UI tasks**, suggest criteria like:
- Component renders correctly on target screen sizes
- Form validation shows appropriate error messages
- Loading and error states handled
- Accessibility requirements met
- Navigation works as expected

**For bug fixes**, suggest criteria like:
- Original bug is no longer reproducible
- No regression in related functionality
- Root cause documented
- Test added to prevent recurrence

Adapt suggestions to the specific ticket — generic criteria are less useful than tailored ones.

### Step 2 — Present as Checklist {#acc.2.present-checklist}

Present the suggested criteria as a numbered checklist:

```
Suggested Acceptance Criteria:

Based on your description, here are the criteria I'd recommend:

  1. ✅ [criterion 1 — specific to this ticket]
  2. ✅ [criterion 2 — specific to this ticket]
  3. ✅ [criterion 3 — specific to this ticket]
  4. ✅ [criterion 4 — specific to this ticket]
  5. ✅ [criterion 5 — specific to this ticket]

All are selected by default. You can remove any that don't apply
and add your own.
```

Use `AskUserQuestion`:

- **Keep all, looks good** — Use all suggested criteria as-is
- **Let me select which ones to keep** — I want to remove some
- **Keep all and add more** — These are good, plus I have additional ones
- **Start fresh** — I want to write my own from scratch

### Step 3 — Handle User Selection {#acc.3.handle-selection}

**If "Let me select which ones to keep":**

Ask: "Which criteria numbers would you like to REMOVE? (e.g., 2, 4)"

Remove those and re-present the remaining list. Then ask:

Use `AskUserQuestion`:

- **This is the final list** — Proceed with these criteria
- **I want to add more criteria** — I have additional ones to include

**If "Keep all and add more":**

Ask: "What additional acceptance criteria would you like to add?"

Collect the additional criteria, format them cleanly, and add to the list. Re-present the complete list:

```
Final Acceptance Criteria:

  1. ✅ [criterion 1 — suggested]
  2. ✅ [criterion 2 — suggested]
  3. ✅ [criterion 3 — suggested]
  4. ✅ [criterion 4 — user added]
  5. ✅ [criterion 5 — user added]
```

Use `AskUserQuestion`:

- **Yes, finalize these** — Proceed with this list
- **I want to add even more** — I have more criteria
- **Let me remove some** — I want to trim the list

Repeat until the user confirms the final list.

**If "Start fresh":**

Ask: "What are your acceptance criteria? List the conditions that must be met for this to be considered done."

Collect, format as clean bullet points, and present back for confirmation.

### Step 4 — Final Confirmation {#acc.4.final-confirmation}

Before moving on, always present the final acceptance criteria list and confirm:

```
Confirmed Acceptance Criteria:

- [criterion 1]
- [criterion 2]
- [criterion 3]
- ...
```

Use `AskUserQuestion`:

- **Yes, these are final** — Lock in these criteria and proceed
- **One more change** — I want to make a small adjustment
