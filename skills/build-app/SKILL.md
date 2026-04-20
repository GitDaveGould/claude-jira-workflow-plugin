---
name: build-app
description: Orchestrate sub-agents to implement Jira tickets — all, a specific epic, or a single story
disable-model-invocation: true
argument-hint: "[TICKET-KEY or EPIC-KEY — omit to build all]"
---

# Build App

## Current State
- Branch: !`git branch --show-current`
- Git status: !`git status --short`

---

Orchestrate implementation of the Jira backlog by spawning sub-agents per ticket.

**Scope modes:**
- `/build-app` — build all To Do stories in the project
- `/build-app PROJ-42` — build a single story
- `/build-app PROJ-E1` — build all To Do stories in one epic

---

## Phase 1: Safety Checks

1. Read `CLAUDE.md` in the current directory. If missing, stop:
   > "CLAUDE.md not found. Run `/init-jira-workflow` then `/plan-app` first."

2. Extract `JIRA_PROJECT_KEY` and `AGENT_MODEL` from `CLAUDE.md`. If `AGENT_MODEL` is missing or unset, default to `sonnet`.

3. Check git status from the dynamic context above. If there are uncommitted changes, stop:
   > "Uncommitted changes detected. Please commit or stash your changes before running build-app."

4. Parse `$ARGUMENTS` to determine scope:
   - **Empty** → scope is `all`
   - **Non-empty** → use `getJiraIssue` to fetch the key and read its `issuetype`:
     - `Story` (or `Task`, `Bug`) → scope is `single`, store the ticket
     - `Epic` → scope is `epic`, store the epic key
     - Anything else → stop: `"PROJ-KEY is a [type] — build-app accepts a Story key or Epic key only."`

---

## Phase 2: Fetch Backlog

Fetch stories based on scope:

**Scope: all**
```
project = PROJECT_KEY AND status = "To Do" AND issuetype = Story ORDER BY created ASC
```

**Scope: epic**
```
project = PROJECT_KEY AND status = "To Do" AND issuetype = Story AND "Epic Link" = EPIC-KEY ORDER BY created ASC
```

**Scope: single**
- No JQL needed — the story was already fetched in Phase 1.
- Confirm the story's status. If it is not "To Do" or "In Progress", stop:
  > "PROJ-42 has status [status] — nothing to build."
- Skip to Phase 4 (single-story execution path).

**If no stories are found (all or epic scope):**
- Also check `status = "In Progress"` — if any exist, they may be stuck
- Report: "No 'To Do' stories found. Check /ticket-status for current board state."

Group the fetched stories by their parent epic for batch ordering.

---

## Phase 3: Batch Ordering

*(Skip this phase for scope: single — go directly to Phase 4.)*

Order stories into execution batches. Stories in the same batch run in parallel; batches run sequentially.

**Default batch order:**
1. **Batch 1:** Project Setup & CI stories (must be first — sets up the foundation)
2. **Batch 2:** Data Model & Migrations + Auth & Authorization (can run in parallel)
3. **Batch 3+:** Feature epics (parallel by default)
4. **Final batch:** Testing & Quality Gates stories

**Override to sequential (same batch → separate batches) if:**
- Story B explicitly mentions Story A's output as input
- Both stories modify the same core file (e.g., main schema file, primary router)
- Story summary/description says "after [other story]" or "depends on [other story]"

Print the plan to the user before executing:

```
# For scope: all
Batch plan (all To Do stories):
  Batch 1 (sequential foundation):  PROJ-1, PROJ-2
  Batch 2 (parallel):               PROJ-3, PROJ-4, PROJ-5
  Batch 3 (parallel):               PROJ-6, PROJ-7
  Batch 4 (sequential):             PROJ-8

Ready to start. Proceed? (yes/no)

# For scope: epic
Batch plan — EPIC-KEY: Epic Name (N stories):
  Batch 1 (parallel):  PROJ-3, PROJ-4
  Batch 2 (sequential): PROJ-5

Ready to start. Proceed? (yes/no)
```

Wait for user confirmation before executing.

---

## Phase 4: Execute Batches

### Single-story path (scope: single)

Confirm with the user before running:

```
Single ticket: PROJ-42 — Story summary here

Ready to start. Proceed? (yes/no)
```

Spawn one Task (sub-agent) using model `AGENT_MODEL` with the standard prompt (see below). Report the result when done, then skip to Phase 5.

### Batch path (scope: all or epic)

For each batch:

**Spawn sub-agents**

For each story in the batch, spawn a Task (sub-agent) using model `AGENT_MODEL` with this prompt:

```
Work Jira ticket TICKET_KEY per the work-ticket skill at ~/.claude/plugins/marketplaces/user-plugins/jira-workflow/skills/work-ticket/SKILL.md.

The project CLAUDE.md at [current directory]/CLAUDE.md has conventions and Jira config.

Ticket to implement: TICKET_KEY
```

Run all tasks in the batch in parallel.

**Wait and report**

After all tasks in the batch complete:

1. Use `searchJiraIssuesUsingJql` to check which tickets moved to Done vs stayed in To Do/In Progress
2. Report results to user:

```
Batch N complete:
  ✓ Done:     PROJ-3 (auth middleware), PROJ-4 (user model)
  ✗ Blocked:  PROJ-5 (see ticket comment for details)

N stories remaining. Continue to next batch? (yes/no)
```

3. Wait for user confirmation before proceeding to the next batch.

---

## Phase 5: Completion Summary

After all work completes (or when user stops):

Use `searchJiraIssuesUsingJql` with `status = "Done"` to get completed count. Scope the query to match the original scope (all project stories, epic stories, or the single ticket).

Print summary:

```
Build session complete.

  Stories completed this session: N
  Stories remaining (To Do):      M
  Stories stuck (In Progress):    K

Suggested next steps:
  gh pr list              — review open PRs
  /ticket-status          — view full board
  /build-app              — continue with remaining tickets
```
