# Technical Deep-Dive Protocol

<!-- ============================================================
     SHARED PROTOCOL: Technical Deep-Dive
     ============================================================
     Used by: jira-epic-orchestrator (Mode 1 Stage 5, Mode 4 adding tasks)
     
     Purpose: Gather detailed technical specifications for each task
     interactively — database schemas, API contracts, UI specs, etc.
     
     These specs become part of the JIRA ticket description AND
     the spec file, so execution modes have everything ready.
     ============================================================ -->

## Section IDs

**Prefix:** `tdd`

| ID | Section |
|---|---|
| `tdd.1.database-work` | Database Work |
| `tdd.2.api-work` | API Work |
| `tdd.3.ui-work` | UI Work |
| `tdd.4.business-logic` | Business Logic |
| `tdd.5.infrastructure` | Infrastructure / DevOps |
| `tdd.6.confirm-specs` | After Each Task's Deep-Dive |

## When to Use

Use this protocol during epic planning (Mode 1/2/4) whenever tasks are being decomposed. Every task gets a deep-dive into whatever technical artifacts it requires.

## Protocol

For each task, identify what technical artifacts are needed and ask the user about them one question at a time.

### Database Work (data storage, models, entities) {#tdd.1.database-work}

- Ask: "What database engine? (Postgres, MySQL, MongoDB, etc.)"
- Ask: "What tables/collections are needed? For each one, what columns/fields, types, and constraints?"
- Ask: "Any foreign keys, indexes, or relationships between tables?"
- Ask: "Any seed data or default values needed?"
- Then generate and present:
  - Database model (entity definitions with types, constraints, relationships)
  - Migration/upgrade script (CREATE TABLE, ALTER TABLE, etc.)
  - Entity relationship diagram (as text)

### API Work (endpoints, services, integrations) {#tdd.2.api-work}

- Ask: "What endpoints are needed? (method, path, purpose)"
- Ask: "What are the request/response payloads? (fields, types, validations)"
- Ask: "Authentication/authorization requirements?"
- Ask: "Rate limiting, pagination, or versioning needs?"
- Then generate and present:
  - API contract (endpoint specs with request/response schemas)
  - Validation rules
  - Error response definitions

### UI Work (frontend, screens, components) {#tdd.3.ui-work}

- Ask: "What screens or components are needed?"
- Ask: "What fields, interactions, and validations?"
- Ask: "Any design requirements or component library to follow?"
- Then generate and present:
  - Component hierarchy
  - State management needs
  - Key interactions and user flows

### Business Logic (rules, calculations, workflows) {#tdd.4.business-logic}

- Ask: "What are the rules or conditions?"
- Ask: "Any edge cases or error scenarios?"
- Then generate and present:
  - Logic flow or decision tree
  - Error handling approach

### Infrastructure / DevOps (deployment, config, CI/CD) {#tdd.5.infrastructure}

- Ask: "What environment or platform? (AWS, GCP, Docker, K8s, etc.)"
- Ask: "Any config, secrets, or environment variables needed?"
- Then generate and present:
  - Infrastructure spec
  - Config templates

### After Each Task's Deep-Dive {#tdd.6.confirm-specs}

Present the generated specs and confirm using Human Confirmation Protocol:

- **Yes, specs are correct** — Save and move to next task
- **No, let me adjust** — Modify the technical details
- **Add more details to these specs** — There's more to include

Repeat for every task before moving to the next story.
