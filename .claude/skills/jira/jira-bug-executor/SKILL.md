---
name: jira-bug-executor
description: >
  Use this skill for any JIRA Bug ticket — whenever the user says "start
  jira-bug-executor", "fix bug", "debug ticket", or when the router dispatches
  a Bug issue. This skill owns the **analysis phase only**: reproduce, trace
  the root cause, confirm the diagnosis with the user, and write the agreed
  analysis back into the JIRA ticket description. It does NOT implement the
  fix. Once the user confirms the diagnosis and the ticket description is
  updated, this skill hands off to `jira-executor`, which performs the actual
  code changes, specs, commits, and JIRA status updates. Prefer this over
  calling `jira-executor` directly whenever the ticket type is Bug, because
  bugs need a root-cause pass before any code is written.
---

# JIRA Bug Executor

Guide the user through **analysing** a JIRA bug ticket step by step, then **hand off the implementation to `jira-executor`**. Every step requires explicit human confirmation before proceeding.

**Scope boundary — read this first.** This skill is responsible for:

1. Understanding the bug from the ticket.
2. Reproducing the issue and tracing the root cause.
3. Getting explicit user confirmation of the diagnosis.
4. Writing the confirmed diagnosis back into the ticket description so the fix-time executor (and any reviewer) has a single, authoritative analysis record.
5. Handing off to `jira-executor` to actually implement the fix.

**This skill does NOT:** transition status to "In Progress", create specs, produce code-to-phrase / code-to-sentence, present implementation plans, edit code files, run git, or post the final JIRA summary comment. Those all happen inside `jira-executor` during the handoff.

## Per-skill rule — Remember answers for the session {#bug.remember}

Every `AskUserQuestion` prompt raised by this skill MUST follow the Human Confirmation Protocol's **Remember Answer for the Session** rule (see `.claude/skills/jira/_shared/references/human-confirmation-protocol.md` — section [`hcp.8.remember-answer`]).

Concretely:

- Every non-destructive question offers a "Remember this answer for the session" variant (Pattern A — sibling option, or Pattern B — single toggle), so the user is never asked the same thing twice in a session.
- When a remembered choice is in effect, print a one-line note (`▸ Using remembered choice for <prompt topic>: <value>`) and skip the question.
- Destructive actions (see `hcp.6.destructive-pattern`) are **never** remembered. The analysis-block description update in Step 5 and the handoff trigger in Step 6 are treated as destructive (they write to the ticket and transfer control) — they are always re-asked.

## Section IDs

**Prefix:** `bug`

When starting each step, display the section ID:

```
▶ [bug.step-N.name] — Step title
```

| ID | Step |
|---|---|
| `bug.step-1.fetch-understand` | Fetch and present bug understanding |
| `bug.step-2.reproduce` | Reproduce the issue |
| `bug.step-3.root-cause` | Trace root cause |
| `bug.step-4.confirm-diagnosis` | Confirm diagnosis with user |
| `bug.step-5.update-ticket-description` | Write analysis back into the ticket description |
| `bug.step-6.handoff-to-executor` | Hand off to `jira-executor` for implementation |

## Flow

### Step 1 — Fetch and present bug understanding {#bug.step-1.fetch-understand}

Fetch the JIRA ticket using the `getJiraIssue` MCP tool. Read the ticket summary, description, steps to reproduce, expected behaviour, actual behaviour, and any acceptance criteria thoroughly.

Present your full understanding back to the user:

```
Ticket:      [KEY] — [Summary]
Type:        Bug
Priority:    [Priority]
Status:      [Current Status]

Bug Summary:
[Rephrase what the bug is — what is broken and what impact it has]

Steps to Reproduce:
1. [step 1]
2. [step 2]
...

Expected Behaviour:
[What should happen]

Actual Behaviour:
[What actually happens]

Acceptance Criteria (for the fix):
- [criterion 1]
- [criterion 2]
- ...
```

If the ticket description is missing steps to reproduce, expected/actual behaviour, or other critical details, flag this explicitly and ask the user to fill in the gaps before proceeding.

Then use `AskUserQuestion` to confirm (per `bug.remember` — include a remember variant):

- **Yes, that's correct** — Your understanding is accurate, proceed
- **Partially correct, let me clarify** — Some parts need correction
- **I want to add more context** — There are additional details you should know

If the user selects **"Partially correct"** or **"add more context"**, collect their input, update your understanding, and present it again. Repeat until confirmed.

### Step 2 — Reproduce the issue {#bug.step-2.reproduce}

Before investigating the root cause, attempt to reproduce the issue:

1. **Identify the affected area** — Based on the bug description, locate the relevant files, modules, and code paths.
2. **Trace the execution path** — Follow the code from the entry point (API endpoint, UI component, event handler, etc.) through to where the bug manifests.
3. **Check for tests** — Look for existing tests that should have caught this bug. Note whether tests exist but are passing (meaning they don't cover the scenario) or are missing entirely.

Present your findings:

```
Reproduction Analysis:

Affected Area:
- [file/module] — [why this is involved]
- [file/module] — [why this is involved]

Execution Path:
[entry point] → [step 1] → [step 2] → ... → [where bug manifests]

Existing Test Coverage:
- [test file] — [covers/doesn't cover this scenario]
- No tests found for [specific scenario]
```

Then use `AskUserQuestion` to confirm:

- **Yes, that matches what I see** — Your reproduction analysis is correct
- **Not quite, let me clarify** — The bug manifests differently than described
- **I can provide more clues** — I have additional debugging info

### Step 3 — Trace root cause {#bug.step-3.root-cause}

Now dig deeper to identify the root cause. This is the critical investigation step:

1. **Read the relevant code carefully** — Don't just skim. Read the functions, their inputs, outputs, and edge cases.
2. **Identify the defect** — What specific line(s), condition(s), or logic error(s) cause the bug?
3. **Understand why it was introduced** — Was it a missing edge case? A wrong assumption? A regression from another change? Check `git log` / `git blame` if helpful.
4. **Assess blast radius** — What else could be affected by this same root cause? Are there similar patterns elsewhere that might have the same bug?

Present your root-cause analysis:

```
Root Cause Analysis:

Defect Location:
- [file:line] — [description of the defect]

Root Cause:
[Clear explanation of WHY the bug happens — not just what is wrong,
but the underlying reason.]

How It Was Introduced:
[If determinable — e.g., "This edge case was not handled in the
original implementation" or "Regression from commit abc123 which
changed the filter logic"]

Blast Radius:
- [Other places that might be affected]
- [Or: "Isolated to this single code path"]
```

Then use `AskUserQuestion` to confirm:

- **Yes, that's the root cause** — Your diagnosis is correct, proceed to fix
- **Close but not exactly** — The root cause is slightly different
- **No, that's wrong** — Let me point you in the right direction

If the user corrects or redirects, re-investigate and present the updated analysis. **Do NOT proceed until the user confirms the root cause.**

### Step 4 — Confirm diagnosis with user {#bug.step-4.confirm-diagnosis}

Present a consolidated diagnosis summary and the proposed fix approach (high level, not code yet):

```
Confirmed Diagnosis:

Bug:          [one-line summary of what's broken]
Root Cause:   [one-line summary of why]
Fix Approach: [high-level description of the fix — e.g., "Add a null check
               before accessing the property" or "Change the query to use
               LEFT JOIN instead of INNER JOIN"]
Blast Radius: [isolated / moderate / wide — with brief explanation]
```

Then use `AskUserQuestion`:

- **Yes, proceed** — Diagnosis confirmed; record it on the ticket and hand off
- **I want to adjust the approach** — The diagnosis is right but I want a different fix strategy
- **Go back and investigate more** — I'm not confident in this diagnosis yet

If the user selects "adjust the approach", iterate on the Fix Approach line and re-ask. If they select "investigate more", loop back to Step 2 or Step 3 as appropriate. **Only advance to Step 5 once the user explicitly confirms "Yes, proceed".**

### Step 5 — Write analysis back into the ticket description {#bug.step-5.update-ticket-description}

Once the diagnosis is confirmed, update the JIRA ticket description so the analysis lives on the ticket itself. This makes the ticket the single source of truth before `jira-executor` takes over.

**Resolve the confirming user's display name.** Before building the analysis block, call the `atlassianUserInfo` MCP tool to fetch the current Atlassian user and use their `displayName` as the **Confirmed with reporter** attribution. If `atlassianUserInfo` is unavailable or returns no usable name, fall back to the ticket's Reporter field (from the `getJiraIssue` response) and, failing that, use the literal string `current user`. Cache the resolved name for the remainder of this skill invocation — do not look it up twice.

Build the updated description by **appending** a clearly-delimited analysis block to the existing description (do not replace the original content — preserve the original reporter text above the block):

```
---
## Analysis (added by jira-bug-executor on [ISO-8601 timestamp])

**Bug:** [one-line summary of what's broken]

**Reproduction:**
[Copy the reproduction analysis from Step 2 — affected area, execution path, test coverage gaps]

**Root Cause:** [one-line summary of why]
Full detail:
[Copy the root-cause analysis from Step 3 — defect location, root cause, how it was introduced]

**Fix Approach:** [high-level description of the fix agreed with the user]

**Blast Radius:** [isolated / moderate / wide, with brief explanation]

**Confirmed with reporter:** yes ([resolved display name] via jira-bug-executor)
---
```

The `[resolved display name]` placeholder MUST be replaced by the real name returned from the resolution step above — never leave the literal string `[user]`, `[resolved display name]`, or any other placeholder in the block that gets written to JIRA.

Before calling the API, show the exact new description block to the user and use `AskUserQuestion`.

**Destructive pattern:** this is a destructive write to the JIRA ticket. Per `hcp.6.destructive-pattern`, the question is **always asked** and never carries a "Remember for session" variant — even if the user has been breezing through earlier prompts.

- **Yes, update the ticket** — Save the analysis to the ticket description
- **Let me tweak the wording** — I want to adjust the analysis text before it's saved
- **Skip the description update** — Hand off to `jira-executor` without updating the description

If the user chooses to tweak, accept their edits and re-present for confirmation. If they choose to skip, note that the description was not updated and proceed to Step 6.

If the user confirms, call `editJiraIssue` with the combined description (original body + the analysis block) to save the change. Confirm back to the user that the ticket has been updated.

Do **NOT** transition the ticket status here — the transition to "In Progress" happens inside `jira-executor` after its Step 7 (Transition to In Progress), per that skill's own rules.

### Step 6 — Hand off to `jira-executor` {#bug.step-6.handoff-to-executor}

The analysis phase is complete. Hand off the ticket to `jira-executor` for the actual fix, specs, commits, and JIRA comment update.

Announce the handoff to the user:

```
▶ [bug.step-6.handoff-to-executor] — Handing off to jira-executor

Analysis recorded on [KEY]. Starting `jira-executor` now — it will:
  • Fetch the ticket (now including the analysis block)
  • Auto-confirm the ticket understanding from the analysis block
  • Produce the Dev Flows (code-to-phrase / code-to-sentence) for the fix
  • Run the Spec-First Check against .spec/dev/
  • Present an implementation plan — every step tied 1:1 to a phrase
  • Implement the fix strictly per the phrase list (no extra files / tests / refactors)
  • Transition the ticket to In Progress once the code is ready
  • Commit and post the final JIRA update
```

Then **read `.claude/skills/jira/jira-executor/SKILL.md` and begin executing its flow from Step 1**, passing the same ticket key. From here on, `jira-executor` is in control.

Do not repeat the ticket understanding, spec check, plan, or code steps inside this skill — that is exactly what `jira-executor` is for, and duplicating them would fork the execution path that the project is trying to keep single.

## Important Principles

- **Analysis here, implementation in `jira-executor`.** This skill's job ends once the ticket description carries the agreed analysis.
- **Never write code in this skill.** The temptation is to apply a small fix inline after diagnosis. Don't — route through `jira-executor` so every bug fix follows the same implementation discipline as every other ticket.
- **The ticket description is the handoff contract.** Because the analysis lives on the ticket, `jira-executor`'s Step 1 fast-path detects the `## Analysis (added by jira-bug-executor on ...)` block and auto-confirms the understanding — no out-of-band context passing is needed.
- **Always confirm before acting.** No description edits and no handoff without explicit user approval. Destructive prompts (Step 5) never carry a remember-for-session option.
