# Code to Phrase Generator {#code-to-phrase}

Convert each planned code change into a single short English phrase. The phrase names the code artefact being added, changed, or removed — nothing about runtime behaviour, no reasoning, no full sentences.

## Rules {#code-to-phrase.rules}

- **One phrase = one code change.** If the phrase describes two unrelated artefacts, split it.
- **Start with a verb in imperative present tense** — `add`, `create`, `update`, `remove`, `move`, `rename`, `wire`, `expose`. Do not use `will`, `should`, `the system will`, or past tense.
- **Say what code you are adding, not what the user or system does at runtime.** The phrase describes the artefact (page, route, input, button, column, field, service call, migration), not the behaviour it later produces.
- **Keep it short.** One line. A comma is fine to list the obvious sub-parts of the same artefact (the inputs a page contains, the buttons on a form).
- **No reasoning, no "why", no acceptance criteria.** Reasons belong in `phrase-to-sentence-generator.md`.
- **Group under a header** when the change spans several artefacts in the same area — e.g., `### Frontend`, `### Backend`, `### DB`.
- **List in execution order.** Read top-to-bottom as the sequence a coder will follow. If change B depends on change A, A comes first. Within a group, order by dependency; across groups, order groups by dependency (e.g., DB before Backend before Frontend when the later layers consume the earlier ones).
- **Include every small change — no omissions.** List plumbing and glue too: route wiring, router registration, imports/exports, config entries, env vars, DTO/schema updates, migration files, feature-flag toggles, and any file rename/move. If it is going to be a code edit, it must appear as a phrase.

## Enforcement during coding {#code-to-phrase.enforcement}

The `code-to-phrase` list is the authoritative checklist for the change. During implementation:

- **Default: follow the list strictly.** Implement exactly what is written, in the listed order. Do not silently extend scope, and do not ask the user for approval on every small detail that is already covered by a listed phrase — just execute.
- **Rare case only — pause and confirm.** If you genuinely discover a required change that is *not* covered by any phrase in the list (a missing wire-up, an unexpected refactor, a dependency bump, a missed input field, a library upgrade, a config toggle, etc.), STOP coding, surface the discovery to the user, and get **explicit confirmation** before proceeding. This is the exception path — use it sparingly, only for real new findings.
- **Update this file first.** Once confirmed, add the new phrase(s) in the correct position of the sequence (preserving execution order), and only then make the change in code.
- **Never apply the change in code ahead of updating the spec.** The spec must remain the single source of truth for what is being built.

## Good {#code-to-phrase.good}

- add forget-password link in sign-in page
- create forget-password page with email input and "send reset link" button; show "reset email sent" confirmation on submit
- add `POST /auth/forgot-password` route accepting email in body
- add migration creating `password_reset_tokens` table with `user_id`, `token_hash`, `expires_at`, `used_at`

## Bad {#code-to-phrase.bad}

- The user clicks the forget-password link and is taken to a new page — *describes runtime UX, not code*
- Will add a new endpoint that accepts email, generates a token, and sends an email — *uses "will", bundles many changes, describes behaviour*
- Users should be able to reset their password — *acceptance criterion, not a code change*
- Forget password feature — *not a phrase, not a change*

## Template {#code-to-phrase.template}

```
### <Area — Frontend / Backend / DB / …>
- <verb> <artefact> <minimal detail needed to identify it>
- ...
```

## Full Example {#code-to-phrase.example}

### Frontend — Forgot Password

- add forget-password link in sign-in page
- create forget-password page with email input and "send reset link" button; show "reset email sent" confirmation on submit
- create reset-password page with new-password and confirm-password inputs and a "reset password" button; redirect to sign-in on success
- show error message on reset-password page when the token is invalid, expired, or already used
