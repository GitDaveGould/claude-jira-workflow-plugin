# jira-workflow

A Claude Code plugin for Jira-driven greenfield app development. Goes from "idea" to "implemented app" via a repeatable plan → Jira → sub-agents loop.

## Requirements

- Claude Code with an active Atlassian integration (MCP)
- GitHub CLI (`gh`) authenticated — used for PR creation
- Git

## Installation

### 1. Clone the repo

```bash
git clone https://github.com/GitDaveGould/claude-jira-workflow-plugin.git \
  ~/.claude/plugins/marketplaces/user-plugins/jira-workflow
```

### 2. Register the plugin in Claude Code settings

Add the plugin to `~/.claude/settings.json` (create it if it doesn't exist):

```json
{
  "plugins": [
    "~/.claude/plugins/marketplaces/user-plugins/jira-workflow"
  ]
}
```

If you already have a `plugins` array, just append the path.

### 3. Restart Claude Code

Close and reopen Claude Code (or start a new session) for the plugin's commands and skills to load.

---

## Workflow

```
1. cd ~/code/new-app && git init
2. /init-jira-workflow           → configures project for Jira-driven dev
3. /plan-app "my app idea"      → interview → Jira epics + stories created
4. /build-app                    → sub-agents work through all tickets
   /build-app PROJ-E1            → build a single epic
   /build-app PROJ-42            → build a single ticket
5. /ticket-status                → check progress anytime
```

---

## Commands & Skills

### `/init-jira-workflow`

One-time project setup. Asks for your Jira project key, Atlassian site URL, and app name, then writes three files into the current directory:

- `.mcp.json` — wires up the Atlassian MCP server so Claude can talk to Jira
- `.claude/settings.local.json` — enables the MCP server for this project
- `CLAUDE.md` — project config file that all sub-agents read before doing anything (project key, site URL, branch/commit conventions, design system config, ticket workflow rules)

Optionally accepts a [Claude Design system](https://claude.ai/design) URL — if provided, design tokens and brand guidelines are pulled into planning and applied during UI implementation.

Also optionally accepts a **sub-agent model** — the model used by every `work-ticket` agent when implementing tickets. If skipped, defaults to `sonnet`. See [Model Selection](#model-selection) below.

---

### `/plan-app [idea]`

Turns an app idea into a structured Jira backlog via a planning interview.

**Phase 1 — Interview.** Asks up to 9 questions covering users, MVP scope, platform, tech constraints, authentication, scale, and (if no design system is configured) visual style, component preferences, and accessibility requirements. All questions are asked upfront before any planning begins.

**Phase 2 — Technical plan.** Produces a full technical plan: architecture overview, data model, API surface, epic breakdown, tech stack decision, design system summary, and risk areas. The plan is presented to the user for approval before any Jira tickets are created — it won't proceed without an explicit sign-off.

**Phase 3 — Jira backlog creation.** Once approved, creates epics and stories in Jira in dependency order (infrastructure first, then data model, auth, feature epics, testing). Each story gets:
- A concise summary
- A description with context
- Acceptance criteria formatted as `- [ ] item` checklist items (minimum 2 per story)
- A link to its parent epic

Stories are sized to be completable by one sub-agent in one session (3–6 stories per epic, split if too large).

**Phase 4 — CLAUDE.md update.** Writes the confirmed tech stack and resolved design system guidelines back into `CLAUDE.md` so all future sub-agents work from the same decisions.

---

### `/build-app [TICKET-KEY or EPIC-KEY]`

Orchestrates implementation of tickets by spawning parallel sub-agents. Accepts an optional argument to scope what gets built:

| Usage | Behavior |
|-------|----------|
| `/build-app` | Builds all To Do stories in the project |
| `/build-app PROJ-42` | Builds a single story |
| `/build-app PROJ-E1` | Builds all To Do stories in one epic |

**Safety checks.** Reads `CLAUDE.md`, confirms git is clean (no uncommitted changes), and resolves the scope before doing anything. If a key is passed, the issue type is fetched to confirm it's a Story or Epic.

**Batch ordering (multi-story modes).** Groups stories into execution batches — stories in the same batch run in parallel, batches run sequentially. The default order is: project setup → data model + auth (parallel) → feature epics (parallel) → testing. Stories are pulled into their own sequential batch when they explicitly depend on another story's output or both modify the same core file.

The batch plan (or single-ticket confirmation) is printed for user review and requires confirmation before execution begins.

**Execution.** For each batch, a sub-agent (`work-ticket`) is spawned per story and all run in parallel. After each batch completes, results are reported (Done vs. Blocked) and the user confirms before the next batch starts. Single-story mode skips batching entirely.

**Completion summary.** Reports total stories completed, remaining, and stuck, with suggested next steps.

---

### `/ticket-status`

Prints a live sprint board summary for the current project — To Do, In Progress, and Done counts with ticket keys and titles. Flags any In Progress tickets that haven't been updated in over 2 hours as potentially stuck, with a prompt to check ticket comments or re-run `/build-app`.

---

### `work-ticket` (sub-agent, not user-invocable)

The workhorse skill invoked by `/build-app` for each ticket. Implements a single Jira story end-to-end:

1. **Read `CLAUDE.md`** — extracts project key, site URL, branch/commit conventions, and design system config before touching anything else.
2. **Fetch ticket details** — uses `getJiraIssue` to pull the full story. If acceptance criteria are missing, comments on the ticket and stops.
3. **Check dependencies** — if the ticket references other tickets as prerequisites, checks their status. If any are not Done, transitions this ticket back to To Do with a blocking comment and exits cleanly.
4. **Fetch transition IDs dynamically** — uses `getTransitionsForJiraIssue` every time. Transition IDs are never hardcoded.
5. **Transition to In Progress** — moves the ticket before writing any code.
6. **Create feature branch** — `feature/TICKET-KEY-short-description`
7. **Explore the codebase** — uses Glob and Grep to understand existing patterns, conventions, test structure, and file layout before writing anything.
8. **Apply design system (UI tickets only)** — if a `DESIGN_SYSTEM_URL` is configured in `CLAUDE.md`, re-fetches the latest guidelines before implementation. Applies color tokens (never hardcoded hex values), typography scale, spacing system, component library rules, and accessibility requirements.
9. **Implement** — writes code to satisfy each acceptance criterion exactly. No gold-plating.
10. **Write and run tests** — if tests fail after two fix attempts, transitions the ticket back to To Do with a comment describing what's failing and exits.
11. **Commit** — stages only the changed files with a properly formatted commit message: `[PROJ-123] imperative description`
12. **Push and create PR** — pushes the branch and opens a PR via `gh pr create` with the ticket key, a summary of what was built, and each AC item verified.
13. **Transition to Done** — only after the PR is created and all AC items are verified.
14. **Add completion comment** — posts the PR URL and an implementation summary back to the Jira ticket.

---

## Model Selection

During `/init-jira-workflow` you can optionally choose which Claude model powers the `work-ticket` sub-agents that implement your tickets. If you skip this, `sonnet` is used.

| Model | Speed | Cost | Best for |
|-------|-------|------|----------|
| `haiku` | Fastest | Lowest | Simple, well-defined tickets with clear AC |
| `sonnet` | Balanced | Moderate | Most projects — handles complexity well **(default)** |
| `opus` | Slowest | Highest | Complex tickets requiring deeper reasoning |

The model is stored as `Agent Model` in `CLAUDE.md` and can be changed at any time by editing that field. `/build-app` reads it fresh on every run, so a change takes effect the next time you run `/build-app`.

Note that model selection only applies to `work-ticket` sub-agents. The model used for `/plan-app`, `/build-app`, and `/ticket-status` is whatever model your current Claude Code session is running.

---

## Design System Integration

If you have a design system configured at [claude.ai/design](https://claude.ai/design), you can provide its shareable API URL during `/init-jira-workflow`. When set:

- `/plan-app` fetches the design guidelines upfront and incorporates them into planning — so design questions are skipped in the interview
- The resolved tokens (colors, typography, spacing, components, accessibility level) are written into `CLAUDE.md` under a `Design System` section
- Each `work-ticket` sub-agent re-fetches the design guidelines before implementing any UI ticket, ensuring tokens are always up-to-date
- Color values are always referenced by token name, never as hardcoded hex values

If no design system URL is provided, `/plan-app` covers design via interview questions and documents the agreed style in `CLAUDE.md`.

---

## Files

```
jira-workflow/
├── README.md
├── commands/
│   └── init-jira-workflow.md
├── skills/
│   ├── plan-app/SKILL.md
│   ├── build-app/SKILL.md
│   ├── work-ticket/SKILL.md
│   └── ticket-status/SKILL.md
└── templates/
    ├── CLAUDE.md
    └── mcp.json
```
