# Human Confirmation Protocol

<!-- ============================================================
     SHARED PROTOCOL: Enhanced Human Confirmation
     ============================================================
     Used by: ALL jira skills
     
     Purpose: Every significant step must give the user rich options
     to confirm, adjust, add details, or pause. The goal is to build
     confidence that things are going smoothly — never rush past
     a step without ensuring the user is comfortable.
     
     This replaces the basic "Yes/No" pattern with a richer set
     of options that always includes "add more details."
     ============================================================ -->

## Section IDs

**Prefix:** `hcp`

| ID | Section |
|---|---|
| `hcp.1.core-principle` | Core Principle |
| `hcp.2.understanding-pattern` | After Presenting Understanding |
| `hcp.3.plan-pattern` | After Presenting a Plan |
| `hcp.4.specs-pattern` | After Generating Specs or Artifacts |
| `hcp.5.implementation-pattern` | After Implementation |
| `hcp.6.destructive-pattern` | Before Any Destructive or Irreversible Action |
| `hcp.7.confidence-checkpoints` | Confidence Checkpoints |
| `hcp.8.remember-answer` | Remember Answer for the Session |

## Core Principle {#hcp.1.core-principle}

Never present just "Yes" and "No" as the only options. Always include a path to add more details or context. The user should feel they are guiding every step, not just rubber-stamping.

## Standard Confirmation Patterns

### After Presenting Understanding (tickets, tasks, requirements) {#hcp.2.understanding-pattern}

Use `AskUserQuestion`:

- **Yes, that's correct** — Your understanding is accurate, proceed
- **Partially correct, let me clarify** — Some parts need correction
- **I want to add more context** — There are additional details you should know

### After Presenting a Plan (implementation plan, task breakdown, story list) {#hcp.3.plan-pattern}

Use `AskUserQuestion`:

- **Yes, proceed with this** — Looks good, go ahead
- **No, let me adjust** — I want to change the approach
- **Looks good but I want to add more details** — I have additional requirements or constraints to include before proceeding

### After Generating Specs or Artifacts (database models, API contracts, etc.) {#hcp.4.specs-pattern}

Use `AskUserQuestion`:

- **Yes, specs are correct** — Save and move on
- **No, let me adjust** — I want to modify something
- **Add more details to these specs** — I have additional fields, constraints, or rules to include

### After Implementation (code written, files changed) {#hcp.5.implementation-pattern}

Use `AskUserQuestion`:

- **Yes, looks good** — I'm satisfied with the changes
- **No, something needs fixing** — I see an issue that needs correction
- **Run it again with modifications** — I want to adjust the approach and re-implement

### Before Any Destructive or Irreversible Action (JIRA creation, git push, status transition) {#hcp.6.destructive-pattern}

Use `AskUserQuestion`:

- **Yes, go ahead** — I confirm this action
- **Wait, let me review first** — I want to check something before committing
- **No, skip this** — Don't perform this action

## Confidence Checkpoints {#hcp.7.confidence-checkpoints}

At key milestones in any flow, insert a confidence checkpoint. This is not a simple "continue?", but a genuine check-in:

```
Quick check before we continue:

We've completed [what was done].
Next up: [what's about to happen].

Everything looking good so far?
```

Use `AskUserQuestion`:

- **Yes, all good — continue** — Everything is on track
- **Hold on, I want to revisit something** — Let me go back and check something
- **Yes, but I want to add something first** — There's additional context before we proceed

### When to Insert Confidence Checkpoints

- After completing a major phase (e.g., all stories defined, all tasks spec'd)
- Before transitioning from planning to execution
- Before the first JIRA ticket creation in a batch
- After the first ticket in a batch (to catch issues early before repeating)
- Before any git push or PR creation

## Remember Answer for the Session {#hcp.8.remember-answer}

**This rule applies to every `AskUserQuestion` raised by any JIRA skill (router, ticket-creator, executor, bug-executor, story-executor, batch-executor, epic-orchestrator, flow-creator, etc.).**

Whenever a question is one the user is likely to be asked again later in the same session — same wording, same set of choices — the question MUST include a way for the user to lock their answer for the rest of the session so the same question is not re-asked.

### How to add the remember option

There are two equivalent ways to surface this. Pick whichever fits the question shape better; do not invent a third.

#### Pattern A — Pair each option with a "for everything in this session" variant

Best when there are only 2–3 real choices and the user might want to repeat any of them. Add a sibling option next to each main choice, prefixed `Remember:`.

Example (storage choice):

- **JIRA comment** — Use this answer for this prompt only
- **Spec file** — Use this answer for this prompt only
- **Remember: always JIRA comment this session** — Don't ask this question again, treat the answer as JIRA comment for the rest of the session
- **Remember: always Spec file this session** — Don't ask this question again, treat the answer as Spec file for the rest of the session
- **Skip this step** — Don't store anything for this ticket

#### Pattern B — Add a single "remember my last answer" toggle

Best when there are many choices or the answer is freeform. Append one extra option:

- **Remember my answer for this session** — Apply the same answer automatically to every future occurrence of this exact question, no re-asking

The skill must then store the chosen answer keyed by the question's prompt + option-set, and on every subsequent identical `AskUserQuestion`, skip the prompt and reuse the stored answer. Briefly note in the output that the remembered answer was applied (one line, e.g. `▸ Using remembered choice: "JIRA comment"`).

### Scope and lifetime

- **Session-scoped only.** Remembered answers are forgotten when the session ends. Do NOT persist them to disk or to memory files.
- **Question-scoped only.** A remembered answer applies only to the exact same question (same prompt, same option set). Different questions are still asked.
- **Override.** The user can unstick a remembered answer at any time by saying "ask me again about <topic>" or "forget my last choice for <topic>". When that happens, drop the remembered answer for that question.
- **Never remember destructive confirmations.** Questions covered by `hcp.6.destructive-pattern` (JIRA creation, git push, status transitions, deleting files, force-pushing, etc.) MUST always be asked fresh. Do not offer a remember option for those.
- **Never remember the meta-decision.** Don't ask "should I remember answers in general?" — the option is opt-in per question.

### When NOT to add a remember option

- Destructive / irreversible actions (see `hcp.6.destructive-pattern`).
- Questions whose right answer depends on context that changes between occurrences (e.g., "Does this ticket understanding look right?" — the ticket changes every time).
- Confidence checkpoints (`hcp.7`) — the whole point is the human pausing.

For everything else (storage choice, format choice, "include code-to-sentence?", "skip this subtask?", "create the spec now?", "use auto-generated commit message?", etc.), the remember option must be available.
