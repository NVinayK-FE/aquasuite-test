<!-- ============================================================
     JIRA EPIC ORCHESTRATOR — MODE 4: UPDATE EPIC
     ============================================================
     Trigger: update epic <EPIC-KEY>
     Purpose: Modify an existing epic's structure — add stories/tasks,
              remove items, edit existing items, or restructure.
              No code is written. JIRA tickets and spec files are updated.
     
     4 Stages:
       Stage 1: Load current state (epic + children + spec file)
       Stage 2: Collect modifications (add/remove/edit/restructure)
       Stage 3: Review modified plan with changes highlighted
       Stage 4: Apply changes to JIRA and spec file
     
     Shared Protocols Used:
       - human-confirmation-protocol.md — all confirmations
       - acceptance-criteria-checklist.md — when adding stories
       - technical-deep-dive.md — when adding tasks
     
     Exit: "Mode 4 ends here. No code is written."
     ============================================================ -->

# Mode 4 — Update Epic

Modify an existing epic's structure: add stories and tasks, remove items, edit existing items, or restructure the hierarchy. JIRA tickets and spec files are updated. **No code is written.**

## Section IDs

**Prefix:** `update`

When starting each stage, display the section ID:

```
▶ [update.N.name] — Stage title
```

| ID | Stage |
|---|---|
| `update.1.load-current-state` | Load Current State |
| `update.2.collect-modifications` | Collect Modifications |
| `update.3.review-modified-plan` | Review Modified Plan |
| `update.4.apply-changes` | Apply Changes to JIRA and Spec File |

---

## Stage 1 — Load Current State {#update.1.load-current-state}

**Goal:** Fetch the epic, load its children (stories/tasks), and display the current hierarchy and spec file.

### Steps

1. **Get epic key from user input**

   The epic key (e.g., `ACD-5`) is provided in the trigger command.

2. **Fetch the epic**

   Use `getJiraIssue` with the epic key. Extract:
   - Epic summary and description
   - Status
   - Any existing acceptance criteria
   - Parent (if part of an initiative)

3. **Fetch all children**

   Use `searchJiraIssuesUsingJql` with:
   ```
   parent = <EPIC-KEY> ORDER BY created ASC
   ```

   For each child, extract:
   - Key, summary, description
   - Type (Story or Task)
   - Status
   - Parent story (for tasks)

4. **Load the spec file**

   Look for `.spec/epics/<epic-name>.md` or `.spec/initiatives/<initiative-name>.md` (depending on hierarchy).

   If found, read it. If not found, note that no spec file exists yet.

5. **Present the current hierarchy**

   Display the epic structure clearly:

   ```
   Current Epic Structure:
   
   Epic: [KEY] — [Summary]
   Status: [status]
   Spec File: [path or "not found"]
   
   Stories:
     ├─ [KEY] — [Summary] [status]
     │  └─ Tasks:
     │     ├─ [TASK-KEY] — [Summary] [status]
     │     └─ [TASK-KEY] — [Summary] [status]
     └─ [KEY] — [Summary] [status]
        └─ Tasks:
           └─ [TASK-KEY] — [Summary] [status]
   ```

   Then use `AskUserQuestion`:

   - **This looks right** — I'm ready to modify
   - **I want to see the spec file details** — Show me what's in the spec
   - **Let me double-check something first** — I need to clarify the current state

   If the user asks to see the spec file, read and display its contents. Then ask again to proceed to Stage 2.

---

## Stage 2 — Collect Modifications {#update.2.collect-modifications}

**Goal:** Gather all modifications the user wants to make, one at a time.

### The Modification Menu

Use `AskUserQuestion` to present the modification options:

- **Add stories** — Create new stories under the epic
- **Add tasks** — Create new tasks under specific stories
- **Remove items** — Delete stories or tasks
- **Edit existing items** — Change summaries, descriptions, or acceptance criteria
- **Restructure** — Move stories/tasks, reorder, split, or merge
- **I'm done modifying** — Move to Stage 3

Loop this menu until the user selects "I'm done modifying."

### If Adding Stories

Ask: "How many new stories do you want to add?" (number or "as many as we discuss")

For each story:

1. **Gather story details**

   Ask one question at a time:
   - "Story summary?" (title/heading)
   - "Story description?" (context, requirements, business value)
   - Any other details relevant to the user's request

2. **Run Acceptance Criteria Checklist Protocol**

   Follow `.claude/skills/jira/_shared/references/acceptance-criteria-checklist.md`:
   - Suggest acceptance criteria based on the story description
   - Let the user select, modify, or add their own
   - Finalize the acceptance criteria list

3. **Confirm before adding to modification list**

   Use `AskUserQuestion`:
   - **Yes, add this story** — Add to the queue
   - **No, let me revise** — Go back and adjust
   - **Yes, and I want to add more details** — Collect additional context before adding

4. **Add to internal tracking** (do not create JIRA yet)

   Keep a list of new stories to be created, with their summaries, descriptions, and acceptance criteria.

### If Adding Tasks

Ask: "Which story should the new task(s) belong to?" (show list of existing stories)

User selects a story.

Ask: "How many new tasks do you want to add?" (number or "as many as we discuss")

For each task:

1. **Gather task details**

   Ask one question at a time:
   - "Task summary?" (title/heading)
   - "Task description?" (requirements, technical scope, expected output)

2. **Run Technical Deep-Dive Protocol**

   Follow `.claude/skills/jira/_shared/references/technical-deep-dive.md`:
   - Ask about technical artifacts needed (database work, API, UI, business logic, etc.)
   - Gather specs one question at a time
   - Generate and present the technical specs (models, schemas, API contracts, etc.)
   - Confirm the specs before moving on

3. **Run Acceptance Criteria Checklist Protocol**

   Follow `.claude/skills/jira/_shared/references/acceptance-criteria-checklist.md`:
   - Suggest acceptance criteria based on the task and specs
   - Let the user select, modify, or add their own
   - Finalize the acceptance criteria list

4. **Confirm before adding to modification list**

   Use `AskUserQuestion`:
   - **Yes, add this task** — Add to the queue
   - **No, let me revise** — Go back and adjust
   - **Yes, and I want to add more details** — Collect additional context before adding

5. **Add to internal tracking** (do not create JIRA yet)

   Keep a list of new tasks to be created, with their summaries, descriptions, technical specs, and acceptance criteria. Link each task to its parent story.

### If Removing Items

Ask: "Which items do you want to remove?" (show the full hierarchy)

For each item the user indicates:

1. **Warn if item is in-progress**

   If the item's status is "In Progress" or "In Review", warn:
   ```
   Warning: [KEY] is currently [status]. Removing it will
   delete the JIRA ticket. Are you sure?
   ```

   Use `AskUserQuestion`:
   - **Remove it anyway** — Delete this item
   - **Keep it** — Don't delete
   - **Let me reconsider** — Cancel the removal

2. **Add to removal list** (do not delete JIRA yet)

   Keep a list of keys to be removed.

### If Editing Existing Items

Ask: "Which item do you want to edit?" (show the full hierarchy by key and summary)

User picks an item by key.

1. **Show current values**

   ```
   Current Values for [KEY]:
   
   Summary:      [current summary]
   Description:  [current description]
   Acceptance Criteria:
   - [criterion 1]
   - [criterion 2]
   - ...
   ```

2. **Ask which field to edit**

   Use `AskUserQuestion`:
   - **Edit summary** — Change the title
   - **Edit description** — Change the description
   - **Edit acceptance criteria** — Modify the criteria
   - **Edit multiple fields** — Make multiple changes
   - **I'm done editing this item** — Move to next item or back to menu

3. **Collect new value**

   Ask the appropriate question based on the field selected.

4. **Update internal tracking** (do not edit JIRA yet)

   Store the edit in a modification list.

5. **Repeat until user selects "I'm done editing this item"**

   Then ask if they want to edit another item or return to the modification menu.

### If Restructuring

Ask: "What would you like to restructure?"

Use `AskUserQuestion`:
- **Move a story or task** — Move it to a different parent
- **Reorder stories or tasks** — Change the sequence
- **Split a story** — Break one story into multiple stories
- **Merge stories** — Combine multiple stories into one
- **I'm done restructuring** — Back to modification menu

**For Move:**
- Ask: "Which item do you want to move?" (show hierarchy)
- User picks by key.
- Ask: "Where should it go?" (show available parents)
- User picks new parent.
- Add to internal tracking.

**For Reorder:**
- Ask: "Which items do you want to reorder?" (stories or tasks?)
- Show the current order with keys and summaries.
- Ask: "What's the new order?" (accept user's sequence)
- Add to internal tracking.

**For Split:**
- Ask: "Which story do you want to split?"
- User picks by key.
- Ask: "How many stories should it become?" (number)
- For each new story, ask for a new summary and description.
- Optionally run Acceptance Criteria Checklist for each new story.
- Add to internal tracking.

**For Merge:**
- Ask: "Which stories do you want to merge?" (show list)
- User picks 2+ stories.
- Ask: "What should the merged story's summary be?"
- Ask: "What should the merged story's description be?"
- Optionally run Acceptance Criteria Checklist for the merged story.
- Add to internal tracking.

---

## Stage 3 — Review Modified Plan {#update.3.review-modified-plan}

**Goal:** Present the updated hierarchy with all changes highlighted, and confirm before proceeding to Stage 4.

### Present Updated Hierarchy

Show the entire modified epic structure with changes clearly marked:

```
Updated Epic Structure:

Epic: [KEY] — [Summary]
Status: [status]

Stories:
  ├─ [KEY] — [Summary] [status]  [marker: ➕ Added / ✏️ Edited / ➖ Removed]
  │  └─ Tasks:
  │     ├─ [TASK-KEY] — [Summary] [status]  [marker]
  │     └─ [TASK-KEY] — [Summary] [status]  [marker]
  └─ [KEY] — [Summary] [status]  [marker]
     └─ Tasks:
        └─ [TASK-KEY] — [Summary] [status]  [marker]

Summary of Changes:
- ➕ Added: [X new stories, Y new tasks]
- ✏️ Edited: [X items with updated details]
- ➖ Removed: [X items deleted]
- 🔀 Moved/Restructured: [X items]
```

### Confirm Before Proceeding

Use `AskUserQuestion`:

- **Yes, apply these changes** — Proceed to Stage 4
- **No, let me make more adjustments** — Back to Stage 2
- **Let me add more details** — Collect additional context before applying
- **Cancel** — End Mode 4 without applying changes

If the user selects **"No, let me make more adjustments"**, go back to Stage 2 and repeat.

If the user selects **"Let me add more details"**, ask: "What additional details would you like to add?" Collect their input, incorporate it, re-present the updated hierarchy, and ask for confirmation again.

---

## Stage 4 — Apply Changes to JIRA and Spec File {#update.4.apply-changes}

**Goal:** Make all modifications permanent: create new JIRA tickets, update existing ones, remove ones marked for deletion, and update the spec file.

### Destructive Action Confirmation

Before proceeding, confirm that the user is ready to make permanent changes:

```
About to apply changes:

- ➕ Create [X new tickets]
- ✏️ Update [X existing tickets]
- ➖ Delete [X tickets]
- Modify spec file: [path]

This action is irreversible. Ready to proceed?
```

Use `AskUserQuestion`:

- **Yes, go ahead** — Apply all changes
- **Wait, let me review first** — Show me the details before committing
- **No, cancel** — End Mode 4 without applying changes

If the user selects **"Wait, let me review first"**, show the detailed changes (new summaries, descriptions, acceptance criteria) and ask again.

### Apply Changes (if confirmed)

1. **Create new JIRA tickets**

   For each new story:
   - Use `createJiraIssue` with:
     - Project: ACD
     - Issue Type: Story
     - Summary: [story summary]
     - Description: [story description + acceptance criteria]
     - Parent: [epic key]
     - Additional fields: [any custom fields]

   Capture the returned story key.

   For each new task:
   - Use `createJiraIssue` with:
     - Project: ACD
     - Issue Type: Task
     - Summary: [task summary]
     - Description: [task description + technical specs + acceptance criteria]
     - Parent: [parent story key — may be newly created]
     - Additional fields: [any custom fields]

2. **Update existing JIRA tickets**

   For each item marked for editing:
   - Use `editJiraIssue` with:
     - Issue key: [KEY]
     - Fields to update: summary, description, acceptance criteria

3. **Delete JIRA tickets (remove)**

   For each item marked for removal:
   - Do NOT use any delete endpoint (most JIRA instances don't support permanent deletion).
   - Instead, use `transitionJiraIssue` to move the ticket to a "Removed" or "Cancelled" status, OR
   - Use `editJiraIssue` to mark it as "Won't Do" with a note: "Removed from epic [DATE]"
   - Add a comment to the ticket: "Removed from [epic key] during update on [DATE]"

4. **Update the spec file**

   Load the existing spec file (or create one if it doesn't exist).

   Update the file to reflect the new/modified stories and tasks:
   - Add sections for newly created stories and tasks
   - Update sections for edited stories and tasks
   - Remove or archive sections for deleted stories and tasks
   - Preserve technical specs (database models, API contracts, etc.)

   Save the updated spec file.

   If the spec file is new, register it in root `CLAUDE.md` under **Registered Specs** with an ID key.

5. **Add comment to epic**

   Use `addCommentToJiraIssue` to add a summary comment to the epic ticket:

   ```
   Updated epic structure on [DATE]:
   
   ➕ Added: [X new stories, Y new tasks]
   ✏️ Edited: [X items with updated details]
   ➖ Removed: [X items deleted]
   
   New tickets: [list of newly created keys]
   Modified tickets: [list of edited keys]
   Removed tickets: [list of removed keys]
   
   Updated spec file: [spec file path]
   ```

### Summary

After all changes are applied, present a summary:

```
Changes Applied Successfully

Epic: [KEY] — [Summary]

✅ Created [X new tickets]:
   - [KEY] — [Summary]
   - [KEY] — [Summary]
   ...

✅ Updated [X existing tickets]:
   - [KEY] — [Summary]
   - [KEY] — [Summary]
   ...

✅ Removed [X tickets]:
   - [KEY] — [Summary]
   - [KEY] — [Summary]
   ...

✅ Updated spec file: [path]

Mode 4 ends here. No code is written.

Next steps:
- Execute the updated epic: use "execute epic <KEY>" to implement pending tickets
- Update and execute: use "update and execute epic <KEY>" to modify further and then implement
```

---

## Reference Protocols

When specific steps say "follow [protocol-name]", read and apply the protocol from:

`.claude/skills/jira/_shared/references/`

- **human-confirmation-protocol.md** — Used in all stages for confirmations
- **acceptance-criteria-checklist.md** — Used when adding stories and editing acceptance criteria
- **technical-deep-dive.md** — Used when adding tasks to gather technical specs
