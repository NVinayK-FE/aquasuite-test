# Git Workflow Protocol

<!-- ============================================================
     SHARED PROTOCOL: Interactive Git Workflow
     ============================================================
     Used by: jira-executor, jira-batch-executor, 
              jira-epic-orchestrator (execute modes)
     
     Purpose: Provide a consistent, interactive git menu after
     code implementation. Ranges from simple (commit + push) to
     full (commit, push, branch, merge, pull, PR).
     
     Each skill specifies which level of git control it provides.
     ============================================================ -->

## Section IDs

**Prefix:** `git`

| ID | Section |
|---|---|
| `git.1.level-basic` | Level 1 — Basic |
| `git.2.level-standard` | Level 2 — Standard |
| `git.3.level-full` | Level 3 — Full |

## Git Menu Levels

### Level 1 — Basic (used by jira-executor) {#git.1.level-basic}

Use `AskUserQuestion`:

- **Yes, commit and push** — Commit and push changes to remote
- **Commit only, don't push** — Save locally but don't push
- **No, skip git** — Leave changes uncommitted

If committing, use message format: `[TICKET-KEY]: [Summary]`

### Level 2 — Standard (used by jira-epic-orchestrator execute modes) {#git.2.level-standard}

Use `AskUserQuestion`:

- **Yes, commit and push** — Commit all changes and push
- **Commit only, don't push** — Save locally but don't push to remote
- **No, continue without committing** — Move to next item

If committing, use message format: `ACD-[STORY-KEY]: [Story summary]`

### Level 3 — Full (used by jira-batch-executor) {#git.3.level-full}

Present a full interactive git menu that **loops** until the user chooses to move on. Multiple operations can be chained.

Use `AskUserQuestion`:

- **Commit changes** — Stage and commit
- **Push to remote** — Push current branch
- **Create new branch** — Create and switch to a new branch
- **Switch branch** — Switch to an existing branch
- **Merge branch** — Merge another branch into current
- **Pull latest** — Pull from remote
- **Create Pull Request** — Create a PR via `gh` CLI
- **Done, move on** — Exit git menu

#### Commit changes

Ask for commit message using `AskUserQuestion`:

- **Auto-generate** — Format: `[TICKET-KEY]: [Summary]`
- **Let me write it** — Provide custom message

Stage and commit. Show result:

```
Committed: [hash]
Message:   [message]
Branch:    [branch]
Files:     [count] changed
```

Return to git menu.

#### Push to remote

Push current branch. Show result:

```
Pushed: [branch] → origin/[branch]
```

Return to git menu.

#### Create new branch

Ask: "What should the branch be named?"

Use `AskUserQuestion`:

- **Branch from current** — Create from [current branch]
- **Branch from main** — Create from main/master
- **Branch from another** — Specify base branch

Create, switch, confirm. Return to git menu.

#### Switch branch

List local branches. If uncommitted changes exist, warn and ask:

Use `AskUserQuestion`:

- **Stash and switch** — Stash changes, switch, unstash later
- **Commit first** — Commit current changes before switching
- **Cancel** — Stay on current branch

#### Merge branch

Ask: "Which branch to merge into [current branch]?"

Perform merge. If conflicts:

```
⚠️  Merge conflicts in:
- [file]
- [file]
```

Use `AskUserQuestion`:

- **Help me resolve** — Show conflicts and help fix
- **Abort merge** — Cancel and return to previous state

If clean merge, show result. Return to git menu.

#### Pull latest

Pull from remote. Handle conflicts same as merge. Return to git menu.

#### Create Pull Request

Ask one at a time:

1. "PR title?" (default: `[TICKET-KEY]: [Summary]`)
2. "Target branch?" using `AskUserQuestion`:
   - **main** — Merge into main
   - **develop** — Merge into develop
   - **Other** — Specify target
3. "Additional description?" (optional)

Create via `gh pr create`. Show result:

```
PR Created:
  Title:  [title]
  Branch: [source] → [target]
  URL:    [url]
```

Return to git menu.
