# Spec-First Protocol

<!-- ============================================================
     SHARED PROTOCOL: Spec-First Execution
     ============================================================
     Used by: jira-executor, jira-batch-executor, jira-epic-orchestrator (execute modes)
     
     Purpose: Before implementing ANY ticket/task, verify that all
     required .spec files and sections exist. If any are missing,
     create them FIRST, register in CLAUDE.md, then implement.
     
     This ensures the spec library is always complete before
     code is written — the spec is the contract, code follows it.
     ============================================================ -->

## Section IDs

**Prefix:** `spec`

| ID | Section |
|---|---|
| `spec.1.identify-required` | Identify Required Specs |
| `spec.2.scan-existing` | Scan Existing Specs |
| `spec.3.handle-missing` | Handle Missing Specs |
| `spec.4.final-confirmation` | Final Spec Confirmation |

## When to Run

Run this protocol **before every implementation step** — whether executing a single ticket (jira-executor), a batch of tickets (jira-batch-executor), or tasks within an epic (jira-epic-orchestrator Mode 3/5).

## Protocol Steps

### Step 1 — Identify Required Specs {#spec.1.identify-required}

Analyze the ticket/task requirements and determine which `.spec` files and sections are needed for implementation. Consider:

- **Database specs** — If the task involves tables, schemas, or models, check for corresponding spec sections
- **API specs** — If the task involves endpoints, check for API contract specs
- **Naming conventions** — If the task creates new files/folders, check for naming rules
- **Architecture specs** — If the task involves new modules or patterns, check for architecture specs
- **Business rules** — If the task involves logic or validation, check for rules specs

### Step 2 — Scan Existing Specs {#spec.2.scan-existing}

Scan all `.spec/` markdown files and cross-reference against the Registered Specs table in the root `CLAUDE.md`.

Present the findings:

```
Spec Coverage Check:

Required for this task:
  ✅ naming-rule.folder — Found in .spec/rules/naming-rules.md
  ✅ api-contract.auth  — Found in .spec/contracts/api-contracts.md
  ⚠️  db-schema.users   — NOT FOUND (needed for user table implementation)
  ⚠️  api-contract.users — NOT FOUND (needed for user endpoints)

Missing specs must be created before implementation can begin.
```

### Step 3 — Handle Missing Specs {#spec.3.handle-missing}

If all specs are present → proceed to implementation.

If any specs are missing, **do NOT proceed to implementation**. Instead:

Use `AskUserQuestion`:

- **Yes, let's create the missing specs now** — Walk me through defining them
- **Skip specs, use ticket description only** — I'll define specs later
- **Show me all available specs** — Let me check if something already covers this

#### If Creating Missing Specs

For each missing spec, gather the details interactively (one question at a time):

1. Ask what belongs in this spec section (rules, contracts, schemas, etc.)
2. Ask for the specific details (columns, types, endpoints, rules, etc.)
3. Generate the spec content and present it for review

Use `AskUserQuestion` to confirm:

- **Yes, this spec is correct** — Save it and move to next missing spec
- **No, let me adjust** — I want to modify the spec content
- **I want to add more details** — There's more to include in this spec

After confirmation:

1. **Create or update the `.spec/` file** with the new section
2. **Register it in root `CLAUDE.md`** under Registered Specs if it's a new file
3. **Present confirmation:**

```
Spec Created:
  File:     .spec/schemas/db-schemas.md
  Section:  db-schema.users
  Registered in CLAUDE.md: ✅

Remaining missing specs: [count]
```

Repeat until all missing specs are created.

### Step 4 — Final Spec Confirmation {#spec.4.final-confirmation}

After all specs are resolved, present the complete spec coverage:

```
All Required Specs Ready:

  ✅ naming-rule.folder — .spec/rules/naming-rules.md
  ✅ api-contract.auth  — .spec/contracts/api-contracts.md
  ✅ db-schema.users    — .spec/schemas/db-schemas.md (just created)
  ✅ api-contract.users — .spec/contracts/api-contracts.md (just created)

Ready to proceed with implementation.
```

Use `AskUserQuestion`:

- **Yes, proceed with implementation** — All specs look good
- **No, I want to review the specs again** — Let me double-check before coding
