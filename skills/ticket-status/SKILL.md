---
name: ticket-status
description: Show sprint board summary — To Do, In Progress, Done counts and stuck ticket alerts
disable-model-invocation: true
---

# Ticket Status

Show the current state of the Jira board for this project.

---

## Step 1: Read Project Config

Read `CLAUDE.md` to get `JIRA_PROJECT_KEY` and `JIRA_SITE_URL`.

If `CLAUDE.md` is missing or has no project key, stop:
> "CLAUDE.md not found or missing Jira config. Run `/init-jira-workflow` first."

---

## Step 2: Fetch Board Data

Run these 3 JQL queries in parallel using `searchJiraIssuesUsingJql`:

**To Do:**
```
project = PROJECT_KEY AND status = "To Do" AND issuetype = Story ORDER BY created ASC
```

**In Progress:**
```
project = PROJECT_KEY AND status = "In Progress" AND issuetype = Story ORDER BY updated ASC
```

**Done:**
```
project = PROJECT_KEY AND status = "Done" AND issuetype = Story ORDER BY updated DESC
```

---

## Step 3: Print Board Summary

Format the output as a readable table:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 PROJECT_KEY Board — YYYY-MM-DD
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

 TO DO (N)
 ─────────
 PROJ-1   Set up project scaffold and CI pipeline
 PROJ-2   Create database schema and run initial migration
 ...

 IN PROGRESS (N)
 ───────────────
 PROJ-5   Implement user authentication  [updated: Xh ago]
 ...

 DONE (N)
 ────────
 PROJ-3   Add environment configuration
 PROJ-4   Initialize git repository with .gitignore
 ...

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Total: N To Do | N In Progress | N Done
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## Step 4: Flag Stuck Tickets

For each ticket currently "In Progress":
- Check the `updated` timestamp
- If the ticket has not been updated in more than **2 hours**, flag it:

```
⚠ Stuck tickets (no activity > 2h):
  PROJ-5  Implement user authentication  (last updated: 3h ago)
          → Check ticket comments for blockers, or re-run /build-app
```

If no stuck tickets, skip this section.

---

## Step 5: Board Link

End with:
```
Board: https://JIRA_SITE_URL/jira/software/projects/PROJECT_KEY/boards
```
