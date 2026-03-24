---
name: plan-app
description: Plan a new app via interview, create Jira epics and stories from the plan
disable-model-invocation: true
argument-hint: [app-idea]
---

# Plan App

Turn an app idea into a full Jira backlog via a structured interview and technical planning process.

App idea: $ARGUMENTS

---

## Phase 1: Read Project Config

Read `CLAUDE.md` in the current directory. Extract:
- `JIRA_PROJECT_KEY` (the project key, e.g., `MYAPP`)
- `JIRA_SITE_URL` (the Atlassian domain)

If `CLAUDE.md` is missing or doesn't contain a Jira project key, stop and tell the user:
> "This project isn't configured yet. Run `/init-jira-workflow` first to set up Jira integration."

---

## Phase 2: Planning Interview

**CRITICAL: Ask all 6 questions before writing any plan. Wait for the user's answers.**

Present this interview to the user:

---
I need to understand your app before planning. Please answer these 6 questions:

**1. Who are the users?**
Who will use this app? (e.g., internal team, public consumers, small business owners)

**2. What's the MVP scope?**
What are the 3-5 core features that make this app useful? What's explicitly out of scope for v1?

**3. What platform/deployment?**
Web app, mobile app, CLI, API-only? Deployed where? (cloud provider, self-hosted, serverless)

**4. Any tech constraints?**
Preferred languages/frameworks? Must integrate with existing systems? Any forbidden tech?

**5. Authentication requirements?**
Public (no auth), simple login, OAuth/SSO, role-based access control, or something else?

**6. Estimated scale?**
How many users in year 1? Any performance-sensitive operations? Data volume expectations?

---

Wait for the user to answer all 6 questions before proceeding to Phase 3.

---

## Phase 3: Technical Plan

Based on the interview answers, draft a technical plan with these sections:

1. **Architecture Overview** — system components and how they connect
2. **Data Model** — key entities and relationships
3. **API Surface** — main endpoints or service interfaces
4. **Epic Breakdown** — 5-8 epics covering the full MVP
5. **Tech Stack Decision** — language, framework, database, auth solution
6. **Risk Areas** — technical risks and mitigation approaches

### Standard Epic Set (adapt to the specific app)

Always include some version of these, adapted to fit:
1. Project Setup & CI
2. Data Model & Migrations
3. Auth & Authorization
4. Core Business Logic
5. API Layer
6. Frontend / UI (only if the app has a UI)
7. Testing & Quality Gates

Present the full technical plan to the user and explicitly ask:
> "Does this plan look right? Reply 'looks good' or 'proceed' to create the Jira backlog, or tell me what to change."

**Hard gate: Do not create any Jira tickets until the user explicitly approves with "looks good", "proceed", "yes", "approve", or equivalent. Ambiguous responses require clarification.**

---

## Phase 4: Create Jira Backlog

Once the user approves the plan, create all tickets in this order:

**Creation order:** Infrastructure → Data Model → Auth → Feature epics → Testing

### For each Epic:

Use `createJiraIssue` with:
- `issueType`: Epic
- `summary`: Epic name (e.g., "Project Setup & CI")
- `description`: What this epic covers and its definition of done

First verify the project exists and get issue type metadata:
- `getVisibleJiraProjects` — confirm project key is valid
- `getJiraProjectIssueTypesMetadata` — get valid issue type IDs for the project

### For each Story within an Epic:

Use `createJiraIssue` with:
- `issueType`: Story
- `summary`: Concise story title
- `description`: Context and background
- `acceptanceCriteria` (or in description if no dedicated field): formatted as:
  ```
  - [ ] criterion one
  - [ ] criterion two
  - [ ] criterion three
  ```
- Link to parent epic

### Story sizing guidance:
- Each story should be completable by one sub-agent in one session
- If a story feels too large, split it into 2-3 smaller stories
- Aim for 3-6 stories per epic
- Each story must have at least 2 acceptance criteria

---

## Phase 5: Update CLAUDE.md

After all tickets are created, update the **Tech Stack** section of `CLAUDE.md` with the confirmed decisions from the technical plan. Replace the `<!-- Filled in by /plan-app ... -->` comment with the actual stack.

---

## Phase 6: Summary

Print a completion summary:

```
✓ Jira backlog created

  Epics:   N
  Stories: M

Board: https://{{JIRA_SITE_URL}}/jira/software/projects/{{JIRA_PROJECT_KEY}}/boards

Epic breakdown:
  {{EPIC_KEY}}: Epic Name (N stories)
  ...

Next step: /build-app to start implementation
```
