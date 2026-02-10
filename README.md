# Creating Agent Teams

A Claude Code skill for deciding when and how to create agent teams - whether a task needs a single agent, parallel subagents, or a coordinated team.

## What This Skill Does

This skill helps you:
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

### Option 1: Clone to Skills Directory

```bash
git clone https://github.com/ZoranSpirkovski/creating-agent-teams.git ~/.claude/skills/creating-agent-teams
```

### Option 2: Add as Git Submodule

If you're managing skills in a dotfiles repo:

```bash
cd ~/.claude/skills
git submodule add https://github.com/ZoranSpirkovski/creating-agent-teams.git
```

## Files

| File | Purpose |
|------|---------|
| `SKILL.md` | Core skill - decision framework, agent roster, communication patterns |
| `research.md` | Reference - model comparisons, team patterns, anti-patterns |
| `agent-prompt-template.md` | Templates for spawning each agent type |

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

## License

MIT
