# Creating Agent Teams

A Claude Code plugin with a skill for deciding when and how to create agent teams - whether a task needs a single agent, parallel subagents, or a coordinated team.

## What This Plugin Does

This plugin provides a skill that helps you:
- **Analyze tasks** to determine the right approach (single agent vs parallel vs team)
- **Select the right model tier** (Haiku/Sonnet/Opus) for each role
- **Choose agent types** (Explore, general-purpose, code-reviewer, etc.)
- **Compose effective teams** with proper communication patterns
- **Avoid common mistakes** like token waste, scope creep, and coordination overhead

## Prerequisites

Agent teams are **experimental and disabled by default**. Enable them in your Claude Code settings:

```json
// ~/.claude/settings.json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

## Official Documentation

For complete details on agent teams, see the official Claude Code documentation:

**[Agent Teams Documentation](https://code.claude.com/docs/en/agent-teams)**

## Installation

### Option 1: Plugin Install (Recommended)

Add this repository as a marketplace and install:

```bash
/plugin marketplace add ZoranSpirkovski/creating-agent-teams
/plugin install creating-agent-teams@ZoranSpirkovski-creating-agent-teams
```

### Option 2: Clone to Plugins Directory

```bash
git clone https://github.com/ZoranSpirkovski/creating-agent-teams.git ~/.claude/plugins/creating-agent-teams
```

### Option 3: Clone to Skills Directory (Legacy)

```bash
git clone https://github.com/ZoranSpirkovski/creating-agent-teams.git ~/.claude/skills/creating-agent-teams
cd ~/.claude/skills/creating-agent-teams
# Move skill files to root for legacy skill loading
mv skills/creating-agent-teams/* .
rmdir skills/creating-agent-teams skills
```

## Plugin Structure

```
creating-agent-teams/
├── .claude-plugin/
│   └── plugin.json              # Plugin metadata
├── skills/
│   └── creating-agent-teams/
│       ├── SKILL.md             # Core skill - decision framework
│       ├── research.md          # Reference - model comparisons, patterns
│       └── agent-prompt-template.md  # Templates for spawning agents
└── README.md
```

## Key Concepts

### Agent Roster

| Agent | Model | Type | Role |
|-------|-------|------|------|
| Orchestrator | opus | general-purpose | Coordinates, never implements |
| Explorer | haiku | Explore | Finds files, patterns, conventions |
| Frontend | sonnet | general-purpose | React/UI implementation |
| Backend | sonnet | general-purpose | Server-side logic |
| API | sonnet | general-purpose | Endpoints, contracts |
| Reviewer | sonnet | code-reviewer | Quality gate |

### Communication Modes

1. **Hub-and-Spoke** (default) - All communication flows through the Orchestrator
2. **Discussion Mode** - Agents debate peer-to-peer for investigation/architecture decisions

### Model Selection

- **Haiku**: Search, grep, boilerplate, data gathering
- **Sonnet**: Implementation, debugging, code review (default)
- **Opus**: Architecture, ambiguous requirements, orchestration

## Usage

The skill triggers when you:
- Ask to "create a team" or "set up agents"
- Mention "agent team" or "multi-agent"
- Request parallel work on a complex task

Example:
```
Create an agent team to implement the user authentication feature.
We need frontend login form, backend auth service, and API endpoints.
```

## Contributing

To submit this plugin to the official Anthropic marketplace:
- [Plugin Directory Submission Form](https://clau.de/plugin-directory-submission)

## License

MIT
