# Jira Executor Rules {#jira-exec}

Follow the rules in each md file under `jira-executor-rules/` for every Jira task.

## Rules {#jira-exec.rules}

- [Spec ID Conventions](./jira-executor-rules/spec-id-conventions.md) — how to name and ID every spec and section
- [Code to Phrase Generator](./jira-executor-rules/code-to-phrase-generator.md) — rules for turning every logical code change into a short English phrase
- [Phrase to Sentence Generator](./jira-executor-rules/phrase-to-sentence-generator.md) — rules for expanding each phrase into a full descriptive paragraph for humans and documentation

## Per-feature dev-flow artefacts {#jira-exec.per-feature-files}

For every actual code change (each JIRA ticket that writes real code), the executor must produce **dev-flow artefacts** for the feature **before** any code is written:

- **`code-to-phrase`** — the ordered phrase list for the change, authored per the [Code to Phrase Generator](./jira-executor-rules/code-to-phrase-generator.md) rules.
- **`code-to-sentence`** — each phrase expanded into a full descriptive paragraph, authored per the [Phrase to Sentence Generator](./jira-executor-rules/phrase-to-sentence-generator.md) rules.

The executor decides **with the user** where each of these artefacts should live. There are two valid storage targets, and the **default is Spec file** unless the user picks otherwise (or has a remembered session choice that says otherwise):

- **Spec file (default)** — written to `.spec/dev-flows/feature/<feature-name>/code-to-phrase.md` and `.spec/dev-flows/feature/<feature-name>/code-to-sentence.md` respectively. Use this when the feature is long-lived, will be referenced after the ticket closes, or when the team treats `.spec/dev-flows/` as the project's permanent design log. This is the project's default because dev-flow artefacts are most useful as a persistent log that outlives the ticket.
- **JIRA comment** — posted via `addCommentToJiraIssue` on the current ticket, with a clear heading (`### code-to-phrase` / `### code-to-sentence`). Use this for one-off fixes, hotfixes, or work whose dev-flow detail does not need to outlive the ticket.

### Mandatory user prompts {#jira-exec.user-prompts}

The executor MUST surface two questions to the user **before** writing the artefacts. Each question is an `AskUserQuestion`, and each follows the **Remember Answer for the Session** rule from `.claude/skills/jira/_shared/references/human-confirmation-protocol.md` ([`hcp.8.remember-answer`]) — i.e. the user can lock the choice for the whole session and not be re-asked for subsequent tickets.

#### Prompt 1 — Where should `code-to-phrase` be stored?

Asked after the executor has drafted the code-to-phrase content and is ready to persist it. The **default is Spec file** — present it first in the option list and mark it `(default)`.

Options (Pattern A — see `hcp.8.remember-answer`):

- **Spec file (default)** — Save to `.spec/dev-flows/feature/<feature-name>/code-to-phrase.md`
- **JIRA comment** — Post the code-to-phrase as a comment on this ticket
- **Remember: always Spec file this session** — Don't ask again; default to Spec file for every subsequent ticket in this session
- **Remember: always JIRA comment this session** — Don't ask again; default to JIRA comment for every subsequent ticket in this session
- **Skip storing for this ticket** — Don't persist the code-to-phrase (rare; only when the user explicitly says it isn't worth keeping)

If the user picks a "Remember: …" option, store the choice for the rest of the session and apply it automatically for every later `code-to-phrase` storage decision in this session. On each automatic application, print a one-line note: `▸ Using remembered choice for code-to-phrase: <JIRA comment | Spec file>`.

#### Prompt 2 — Generate `code-to-sentence`, and where should it be stored?

Asked **after** the code-to-phrase has been confirmed (the sentence file is a 1:1 expansion of the phrase file, so this question is meaningless until the phrases exist). The **default is Generate and store in Spec file**.

Options:

- **Generate and store in Spec file (default)** — Expand each phrase into `.spec/dev-flows/feature/<feature-name>/code-to-sentence.md`
- **Generate and store in JIRA comment** — Expand each phrase into a paragraph and post as a comment on this ticket
- **Skip code-to-sentence for this ticket** — Don't generate the sentence expansion at all
- **Remember: always Spec file this session** — Always generate, always store in Spec file, don't ask again
- **Remember: always JIRA comment this session** — Always generate, always store in JIRA comment, don't ask again
- **Remember: always skip this session** — Never generate code-to-sentence for the rest of the session, don't ask again

If the user picks a "Remember: …" option, apply it automatically on every later `code-to-sentence` decision and print: `▸ Using remembered choice for code-to-sentence: <JIRA comment | Spec file | Skip>`.

### Rules that apply regardless of where the artefacts live

- **`<feature-name>`** is kebab-case and matches the feature as it appears elsewhere (JIRA ticket summary, code paths, existing specs). Follow the project [naming conventions](../../.spec/dev/rules/naming-rules.md). The feature-name is required for the spec-file path; for JIRA-comment storage it still feeds the heading inside the comment so the artefact is self-identifying.
- **Phrase first, sentence second.** Always produce code-to-phrase completely (and let the user review it) before touching code-to-sentence. Sentences expand phrases 1:1 — they cannot exist without their source phrases.
- **Keep both artefacts in sync.** If a new phrase is added during coding (per the enforcement rule in [`code-to-phrase-generator.md`](./jira-executor-rules/code-to-phrase-generator.md) — i.e., a genuinely new finding confirmed by the user), add the matching descriptive paragraph to the code-to-sentence artefact in the same change. The two artefacts must stay coherent at every commit / comment update.
  - If they live in spec files, the update is a file edit + commit.
  - If they live in JIRA comments, the update is either a follow-up comment (preferred for auditability) or an edit to the original comment.
  - If one lives in a spec file and the other in a JIRA comment (a mixed choice the user is allowed to make), keep both in lock-step regardless.
- **Spec-first, code-second.** No code is written until both artefacts exist (or the user has explicitly opted to skip code-to-sentence via Prompt 2). This is the concrete realisation of the "update the spec first" clause in [`code-to-phrase-generator.md`](./jira-executor-rules/code-to-phrase-generator.md). When the chosen storage is JIRA comment, "exists" means the comment has been posted on the ticket and `addCommentToJiraIssue` returned success.
- **`.spec/` ticket note.** When the artefacts are stored as spec files, they sit under `.spec/`, which normally requires its own ticket. When produced by an executor as part of implementing the current ticket, no separate ticket is needed — they are part of the ticket's deliverable. Creating them outside of an executor flow (e.g., back-filling docs for an already-shipped feature) still requires a ticket. JIRA-comment storage carries no ticket implication of its own.
- **Default location for unanswered prompts is unsafe.** Never silently default to one storage location — always raise Prompts 1 and 2 (or honour the user's remembered session choice). The only acceptable "no prompt" path is when a remembered choice is in effect. The **project default** is `.spec/dev-flows/...` (Spec file) — it is listed first and marked `(default)` in each prompt so a quick "Yes / Enter / accept default" reliably picks the spec file.

## Strict scope — "no extra code" rule {#jira-exec.strict-scope}

Once the code-to-phrase is confirmed by the user, it is the **authoritative, exhaustive checklist** of what this ticket will change. Nothing beyond that list is allowed during implementation. This rule is enforced at three checkpoints inside `jira-executor`:

1. **Step 5 — Implementation Plan gate.** Every file/action/snippet in the plan MUST map 1:1 back to a phrase in `code-to-phrase`. The executor MUST show, for each plan item, the phrase it traces to (e.g., `(phrase: "add forget-password link in sign-in page")`). If any plan item has no matching phrase, it is out of scope and MUST be removed from the plan — either drop it, or go back to Step 2 and extend the phrase list with explicit user confirmation (per `code-to-phrase.enforcement`) before it can appear in the plan.
2. **Step 6 — Implementation gate.** The executor implements exactly the phrases in the confirmed order. It does NOT add:
   - helper files, convenience stubs, `.gitkeep`, `.gitignore`, `README.md`
   - extra tests that are not named in a phrase
   - refactors of nearby code that a phrase does not call out
   - extra imports, configs, env vars, feature flags, CI tweaks that are not named in a phrase
   - "while we're here" tidy-ups
   Any such addition, even if it looks harmless, is treated as out-of-scope and NOT written. If during coding the executor genuinely discovers a required change not covered by any phrase, it pauses, surfaces the discovery, and only proceeds after the user confirms — at which point the code-to-phrase is updated first (per `code-to-phrase.enforcement`) and only then the code.
3. **Post-implementation reconciliation.** After Step 6 the executor MUST verify that every changed file / added artefact traces back to a phrase, and that no phrase was silently skipped. If either check fails, the executor surfaces the discrepancy before moving to Step 7.

The purpose of this rule is simple: the spec (code-to-phrase) is the contract. If it isn't in the spec, it isn't in the code. If it needs to be in the code, it must be in the spec first.
