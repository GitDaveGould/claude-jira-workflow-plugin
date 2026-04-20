# {{APP_NAME}}

## Jira Config

- **Project Key:** {{JIRA_PROJECT_KEY}}
- **Site URL:** {{JIRA_SITE_URL}}
- **Board JQL:** `project = {{JIRA_PROJECT_KEY}} ORDER BY created ASC`
- **In Progress JQL:** `project = {{JIRA_PROJECT_KEY}} AND status = "In Progress" ORDER BY updated ASC`
- **Todo JQL:** `project = {{JIRA_PROJECT_KEY}} AND status = "To Do" ORDER BY created ASC`
- **Agent Model:** {{AGENT_MODEL}}

## Design System

- **URL:** {{DESIGN_SYSTEM_URL}}
- **Guidelines:** <!-- Populated by /plan-app after fetching or interviewing -->

## Ticket Workflow States

Tickets move: **To Do → In Progress → Done**

**IMPORTANT:** Always fetch transition IDs dynamically via `getTransitionsForJiraIssue`. Never hardcode transition IDs — they vary per project.

```
1. Before starting work: transition ticket to "In Progress"
2. After PR merged / AC verified: transition ticket to "Done"
3. If blocked: transition back to "To Do" with a comment explaining the blocker
```

## Acceptance Criteria Format

All Jira stories must have acceptance criteria in this format:

```
- [ ] criterion one
- [ ] criterion two
- [ ] criterion three
```

A ticket is only done when every `- [ ]` line can be marked `- [x]`. Do not transition to Done if any AC items are unverified.

## Commit / Branch Conventions

**Branch naming:**
```
feature/{{JIRA_PROJECT_KEY}}-123-short-description
```

**Commit messages:**
```
[{{JIRA_PROJECT_KEY}}-123] imperative description of what was done
```

Examples:
- `[MYAPP-7] add user authentication endpoint`
- `[MYAPP-12] implement todo list CRUD operations`

## Sub-Agent Rules

When `work-ticket` sub-agents are working on tickets, they must:

1. Read `CLAUDE.md` first, always — before touching any code
2. Fetch ticket details with `getJiraIssue` — if AC is missing, comment "Missing acceptance criteria" and stop
3. Fetch transition IDs dynamically with `getTransitionsForJiraIssue` — never hardcode
4. Transition to **In Progress** before writing any code
5. Create a feature branch: `git checkout -b feature/TICKET-KEY-slug`
6. Explore codebase with Glob/Grep before writing anything
7. Implement only what the AC specifies — nothing more, nothing less
8. Write/update tests. Run them. If still failing after 2 fix attempts → comment blocker, transition back to To Do, exit
9. Commit with proper format: `[PROJ-123] description`
10. Push branch and create PR via `gh pr create`
11. Transition to **Done** only after PR is created and all AC verified
12. Add completion comment with PR link and implementation summary

**Block handling:** If a dependency ticket is not done, transition to To Do and comment: `"Blocked: requires [OTHER-TICKET] — [specific reason]"`, then exit cleanly.

## Tech Stack

<!-- Filled in by /plan-app after planning interview -->

