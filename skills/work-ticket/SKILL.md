---
name: work-ticket
description: Implement a single Jira ticket end-to-end. Invoked by build-app sub-agents. Do not invoke directly.
user-invocable: false
---

# Work Ticket

Implement a single Jira ticket completely, from transition to PR creation.

Ticket key: $ARGUMENTS

---

## Rules

- Follow every step in order. Do not skip steps.
- Read `CLAUDE.md` first, always — before any other action.
- Implement only what the acceptance criteria specify. Nothing more, nothing less.
- Fetch transition IDs dynamically every time. Never hardcode them.

---

## Step 1: Read CLAUDE.md

Read `CLAUDE.md` in the current directory. Extract:
- `JIRA_PROJECT_KEY`
- `JIRA_SITE_URL`
- Branch naming convention
- Commit message format
- Any tech stack or architectural decisions

If `CLAUDE.md` is missing, stop: output "ERROR: CLAUDE.md not found. Cannot proceed without project config."

---

## Step 2: Fetch Ticket Details

Use `getJiraIssue` to fetch the full ticket for `$ARGUMENTS`.

Check for acceptance criteria. They should be in a `- [ ] item` format either in:
- A dedicated "Acceptance Criteria" field
- The description field

**If acceptance criteria are missing or empty:**
- Use `addCommentToJiraIssue` to add: `"Missing acceptance criteria — cannot implement without knowing the definition of done."`
- Stop. Do not proceed.

Note the ticket's:
- Summary (title)
- Description
- Acceptance criteria items
- Any linked tickets (potential dependencies)

---

## Step 3: Check Dependencies

If the ticket description or AC mentions other tickets as prerequisites:
- Use `getJiraIssue` to check each dependency's status
- If any prerequisite is still "To Do" or "In Progress":
  - Use `getTransitionsForJiraIssue` to get transition IDs
  - Transition this ticket back to **To Do**
  - Use `addCommentToJiraIssue` to add: `"Blocked: requires [TICKET-KEY] to be completed first — [specific reason]"`
  - Stop. Exit cleanly.

---

## Step 4: Get Transition IDs

Use `getTransitionsForJiraIssue` with the ticket key.

Find the transition IDs for:
- "In Progress" (or equivalent)
- "Done" (or equivalent)

Store these for use in Steps 5 and 12. Never hardcode transition IDs.

---

## Step 5: Transition to In Progress

Use `transitionJiraIssue` to move the ticket to **In Progress**.

---

## Step 6: Create Feature Branch

```bash
git checkout -b feature/TICKET-KEY-short-description
```

Where:
- `TICKET-KEY` is the full ticket key (e.g., `MYAPP-7`)
- `short-description` is a 2-4 word slug from the ticket summary (kebab-case, lowercase)

Example: `feature/MYAPP-7-user-auth-endpoint`

---

## Step 7: Explore Codebase

Before writing any code, use Glob and Grep to understand the existing codebase:

- Find relevant existing files (similar features, shared utilities, tests)
- Read key files to understand patterns, conventions, imports
- Note the test framework and how tests are structured
- Identify where new code should live

Do not write any code during this phase.

---

## Step 8: Implement

Write the code to satisfy each acceptance criterion.

Guidelines:
- Follow the patterns and conventions found in Step 7
- Implement only what the AC specifies — no gold-plating
- Write clean, readable code following existing project conventions
- Place files in the correct locations per project structure

---

## Step 9: Write and Run Tests

Write or update tests to verify the implementation.

Run the tests. If tests fail:
1. Fix the issue and run again (attempt 1)
2. Fix the issue and run again (attempt 2)

If tests still fail after 2 fix attempts:
- Use `getTransitionsForJiraIssue` to get transition IDs
- Transition ticket back to **To Do**
- Use `addCommentToJiraIssue` to add: `"Blocked: tests failing after 2 fix attempts — [describe what's failing and why]"`
- Stop. Exit cleanly.

---

## Step 10: Commit

Stage only the files changed for this ticket:

```bash
git add [specific files]
git commit -m "[TICKET-KEY] imperative description of what was implemented"
```

Example: `git commit -m "[MYAPP-7] add JWT authentication middleware"`

Do not use `git add .` or `git add -A` — add files explicitly.

---

## Step 11: Push and Create PR

```bash
git push -u origin feature/TICKET-KEY-short-description
```

Then create the PR:

```bash
gh pr create \
  --title "[TICKET-KEY] ticket summary" \
  --body "Closes TICKET-KEY

## What was built
[Brief description of implementation]

## Acceptance Criteria
[List each AC item and confirm it's satisfied]"
```

Note the PR URL from the output.

---

## Step 12: Transition to Done

Use `transitionJiraIssue` to move the ticket to **Done**.

---

## Step 13: Add Completion Comment

Use `addCommentToJiraIssue` to add a completion comment:

```
Implemented and merged.

PR: [PR URL]

Implementation summary:
- [what was built, key decisions made]
- [any noteworthy technical details]

All acceptance criteria verified:
- [x] criterion one
- [x] criterion two
- [x] criterion three
```
