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

## Workflow

```
1. cd ~/code/new-app && git init
2. /init-jira-workflow        → configures project for Jira-driven dev
3. /plan-app "my app idea"   → interview → Jira epics + stories created
4. /build-app                → sub-agents work through all tickets
5. /ticket-status            → check progress anytime
```

## Commands

| Command | Description |
|---------|-------------|
| `/init-jira-workflow` | One-time project setup — writes `.mcp.json`, `CLAUDE.md`, `.claude/settings.local.json` |

## Skills

| Skill | Invocable By | Description |
|-------|-------------|-------------|
| `/plan-app` | User only | Interview → technical plan → Jira backlog creation |
| `/build-app` | User only | Orchestrator that spawns sub-agents per ticket |
| `/ticket-status` | User only | Sprint board summary view |
| `work-ticket` | Claude only | Sub-agent ticket implementation |

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
