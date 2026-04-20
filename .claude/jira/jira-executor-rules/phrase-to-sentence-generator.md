# Phrase to Sentence Generator {#phrase-to-sentence}

Expand every phrase from `code-to-phrase-generator.md` into a **full descriptive paragraph** that a person reading the spec — a teammate, a reviewer, a future maintainer — can fully understand without needing the code or the original ticket. The paragraph keeps the code-change meaning of the phrase but adds every detail a reader needs: what the artefact is, where it sits, what it contains, how it behaves on the main path and on failure, any constraints (validation, auth, accessibility), and the reason it exists.

## Rules {#phrase-to-sentence.rules}

- **One phrase → one paragraph.** Keep the 1:1 mapping with the code-to-phrase list. If a phrase cannot be captured in a single paragraph because it is too big, split the phrase in `code-to-phrase-generator.md` first and expand each half separately.
- **Lead with the artefact.** Start with the same artefact the phrase names (page, route, button, migration, …). Do not re-frame as runtime user behaviour or as "we will add …".
- **Cover the full picture.** A good description should address most of:
    - **What** the artefact is (component, route, field, migration, …)
    - **Where** it sits in the app (which page, which module, which table)
    - **What it contains / looks like** (fields, copy, styling cues, placement)
    - **How it behaves** on the main/success path (what the submit/click/call does end-to-end)
    - **How it behaves** on failure, edge, or empty cases
    - **Constraints** — auth required, validation rules, accessibility, rate limits, analytics, error-response contracts
    - **Why** it exists — a short "so that …" / "because …" clause capturing the reason
- **Use prose paragraphs, not bullets inside the description.** Multiple sentences are expected. Keep the prose tight — no fluff, no marketing language.
- **Present tense, third person.** "The sign-in page exposes …", not "We will add …" or "The user will click …".
- **Keep order and grouping identical to code-to-phrase-generator.md.** Same section headers (`### Frontend`, `### Backend`, `### DB`), same sequence, same phrase wording on the first line of each entry.
- **Do not invent new code.** Every description must correspond to a phrase already in `code-to-phrase-generator.md`. If you find yourself needing to describe something that isn't in the phrase list, stop — that is a signal the phrase list is incomplete, and you should update `code-to-phrase-generator.md` first (per its enforcement rule).
- **Keep in sync with `code-to-phrase-generator.md`.** When the phrase list changes, update this file in the same commit so the two stay coherent.

## Structure {#phrase-to-sentence.structure}

Each entry is a two-part bullet: the phrase from `code-to-phrase-generator.md` on the first line, and the full descriptive paragraph on the second (indented).

```
- <phrase from code-to-phrase>
  - <full-description paragraph: what + where + contents + main-path behaviour + failure behaviour + constraints + reason>
```

## Good {#phrase-to-sentence.good}

- add forget-password link in sign-in page
  - The sign-in page exposes a "Forgot password?" link positioned directly under the password input, rendered in the same visual style as other secondary links on the page. Clicking the link navigates the user to the dedicated forget-password page without submitting the sign-in form. The link is always visible to unauthenticated visitors, requires no network call of its own, and is keyboard-focusable for accessibility. It exists so that a user who cannot remember their password can start the account-recovery flow directly from the sign-in surface, without contacting support or digging through help pages.

## Bad {#phrase-to-sentence.bad}

- add forget-password link in sign-in page
  - A link is added to the sign-in page. — *Too terse. No placement, no behaviour, no reason.*
- add forget-password link in sign-in page
  - When the user clicks the link they are taken to a new page. — *Describes runtime behaviour from the user's perspective and says nothing about the artefact being added, its placement, or the reason.*
- add forget-password link in sign-in page
  - The sign-in page exposes a forget-password link that routes to the forget-password page. We should also track the click event for analytics, make sure the link is accessible, add a second "Need help?" link beside it, and consider A/B testing the copy. — *Bundles changes that are not in the phrase list. If analytics/accessibility/second-link belong in the work, add them as separate phrases in `code-to-phrase-generator.md` first.*

## Template {#phrase-to-sentence.template}

```
### <Area — Frontend / Backend / DB / …>
- <phrase from code-to-phrase>
  - <paragraph: what the artefact is, where it sits, what it contains, how it behaves on success and failure, any constraints, and the reason it exists>
- ...
```

## Full Example {#phrase-to-sentence.example}

Mirrors the Full Example in `code-to-phrase-generator.md`.

### Frontend — Forgot Password

- add forget-password link in sign-in page
  - The sign-in page exposes a "Forgot password?" link positioned directly under the password input, rendered in the same visual style as other secondary links on the page. Clicking the link navigates the user to the dedicated forget-password page without submitting the sign-in form. The link is always visible to unauthenticated visitors, requires no network call of its own, and is keyboard-focusable for accessibility. It exists so that a user who cannot remember their password can start the account-recovery flow directly from the sign-in surface, without contacting support.

- create forget-password page with email input and "send reset link" button; show "reset email sent" confirmation on submit
  - The forget-password page is a standalone, unauthenticated page reached from the sign-in page link. It contains a single email input with standard email-format validation, a "Send reset link" primary button, and a "Back to sign-in" secondary link. On submit the page disables the button, calls the backend `POST /auth/forgot-password` endpoint with the entered email, and — regardless of whether the email matched an account — replaces the form with a uniform "Reset email sent — check your inbox" confirmation. Resubmitting from the confirmation is blocked until the page is reloaded. The uniform response prevents the UI from leaking which emails correspond to real accounts, and the dedicated page gives the recovery flow enough room to handle loading and confirmation states clearly.

- create reset-password page with new-password and confirm-password inputs and a "reset password" button; redirect to sign-in on success
  - The reset-password page is the landing page opened when a user clicks the tokenized link in their reset email. It reads the token from the URL query string and does not render the form if the token is missing or malformed. When the token is present, the page shows a new-password input, a confirm-password input that must match the new-password value, and a "Reset password" primary button. On submit the page calls the backend `POST /auth/reset-password` with the token and new password; on success it shows a brief "Password updated" message and redirects to the sign-in page so the user can immediately log in with their new credentials. The redirect keeps the recovery flow short and gives the user a clear next step.

- show error message on reset-password page when the token is invalid, expired, or already used
  - When the backend rejects the reset call with any of the invalid-token, expired-token, or already-used-token conditions, the reset-password page shows a single uniform error message ("This reset link is no longer valid. Please request a new one.") near the form and keeps a "Request a new reset email" link visible so the user can re-enter the flow from the forget-password page. The three underlying conditions are intentionally collapsed into one message so the UI does not reveal which specific state the token was in, matching the backend's uniform-error contract and preventing information leaks to an attacker holding a partially valid token.
