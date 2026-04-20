---
description: Configure project for Jira-driven development
disable-model-invocation: true
allowed-tools: Read, Write, Bash(ls:*), Bash(mkdir:*)
---

# Initialize Jira Workflow

Set up this project directory for Jira-driven development using sub-agents.

## Step 1: Guard Check

Check if `.mcp.json` already exists in the current directory:

```
Bash: ls -la .mcp.json 2>/dev/null
```

If `.mcp.json` already exists, **stop immediately** and tell the user:
> "This project is already configured for Jira workflow (`.mcp.json` exists). Run `/plan-app` to start planning, or `/ticket-status` to check the board."

## Step 2: Gather Information

Ask the user for the following (1-3 required, 4-5 optional):

1. **Jira project key** — the short uppercase key for your project (e.g., `MYAPP`, `TODO`, `SHOP`)
2. **Jira site URL** — your Atlassian domain (e.g., `mycompany.atlassian.net`)
3. **App name** — human-readable name for this project (e.g., `My Todo App`)
4. **Claude Design system URL** *(optional)* — if you have a design system set up at claude.ai/design, provide the shareable API URL here. Press Enter to skip.
5. **Sub-agent model** *(optional)* — the model used by each `work-ticket` sub-agent when implementing tickets. Options:
   - `haiku` — fastest and cheapest; best for simple, well-defined tickets
   - `sonnet` — balanced capability and cost; handles most tickets well **(default if skipped)**
   - `opus` — most capable; best for complex tickets requiring deeper reasoning, at higher cost and slower speed

   Press Enter to use the default (`sonnet`).

Wait for the user to provide values 1-3 before proceeding. If they skip question 4, set `DESIGN_SYSTEM_URL` to `none`. If they skip question 5, set `AGENT_MODEL` to `sonnet`.

## Step 3: Write Configuration Files

Once you have the three values, create the following files in the **current working directory**:

### `.mcp.json`

Copy exactly from the template:

```json
{
  "mcpServers": {
    "atlassian": {
      "type": "sse",
      "url": "https://mcp.atlassian.com/v1/sse"
    }
  }
}
```

### `.claude/settings.local.json`

Create `.claude/` directory if it doesn't exist, then write:

```json
{
  "enableAllProjectMcpServers": true,
  "enabledMcpjsonServers": ["atlassian"]
}
```

### `CLAUDE.md`

Read the template from `~/.claude/plugins/marketplaces/user-plugins/jira-workflow/templates/CLAUDE.md` and substitute:

- `{{APP_NAME}}` → the app name the user provided
- `{{JIRA_PROJECT_KEY}}` → the project key (uppercase)
- `{{JIRA_SITE_URL}}` → the site URL (without `https://`)
- `{{DESIGN_SYSTEM_URL}}` → the design system URL, or `none` if skipped
- `{{AGENT_MODEL}}` → the chosen model (`haiku`, `sonnet`, or `opus`), or `sonnet` if skipped

Write the substituted content to `CLAUDE.md` in the current directory.

## Step 4: Confirm Success

After writing all three files, tell the user:

```
✓ Project configured for Jira workflow

Files created:
  .mcp.json                      — Atlassian MCP server config
  .claude/settings.local.json    — Enables MCP in this project
  CLAUDE.md                      — Project conventions and Jira config

Project: {{APP_NAME}} ({{JIRA_PROJECT_KEY}})
Jira:    https://{{JIRA_SITE_URL}}/jira/software/projects/{{JIRA_PROJECT_KEY}}/boards
Agents:  {{AGENT_MODEL}}

Next steps:
  /plan-app "describe your app idea"   → interview + create Jira backlog
  /ticket-status                        → view board anytime
```

**Note:** Reload Claude Code (or open a new session) for the Atlassian MCP server to connect.
