---
name: creating-agent-teams
description: Use when facing a complex task that could benefit from multiple agents, parallel work, or specialized roles - triggers on "create a team", "parallelize", "multi-agent", or cross-layer implementation needs
---

# Creating Agent Teams

## Overview

This skill helps you decide whether a task needs a single agent, parallel subagents, or a coordinated team - and compose effective teams with the right models, agent types, and communication patterns.

**Core principle:** Don't create a team when a single sonnet agent could do it in one pass. Teams add coordination overhead - use them only when parallelism or specialization provides real benefit.

**Official documentation:** [Agent Teams on code.claude.com](https://code.claude.com/docs/en/agent-teams)

## Prerequisites

Agent teams are experimental and disabled by default. Ensure this is enabled before creating any team:

```json settings.json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

## Agent Roster

Predefined agents the Orchestrator picks from based on task needs. Only spawn agents whose scope is actually touched.

| Agent | Model | Type | Role | Default Scope |
|-------|-------|------|------|---------------|
| **Orchestrator** | opus | general-purpose | Plans, delegates, synthesizes. Never implements directly. | Full project awareness via Explorer |
| **Explorer** | haiku | Explore | Answers Orchestrator's questions about the codebase. Finds files, patterns, conventions. | Targeted dirs only — search what's asked |
| **Frontend** | sonnet | general-purpose | React components, views, styling, state management | `frontend/` only |
| **Backend** | sonnet | general-purpose | Rails controllers, models, services, business logic | `app/`, `lib/`, `config/` |
| **API** | sonnet | general-purpose | Endpoints, serializers, shared contracts across apps | API-related files across all apps |
| **iOS** | sonnet | general-purpose | Swift/SwiftUI, MVVM architecture | `mobile/ios/` only |
| **Android** | sonnet | general-purpose | Kotlin/Compose implementation | `mobile/android/` only |
| **Database** | sonnet | general-purpose | Migrations, schema design, query optimization | `db/`, model files |
| **Debugger** | sonnet | general-purpose | Writes tests, debugs failures, validates implementations | Scoped to failing area |
| **Copywriter** | haiku | general-purpose | User-facing text, labels, error messages, i18n strings | String/locale files only |
| **Reviewer** | sonnet | code-reviewer | Quality gate. Reviews against spec and coding standards. | Files changed by other agents |

**Rules:**
- Explorer and Reviewer are always present for any implementation task
- Only spawn agents whose scope is actually touched
- If a task doesn't fit any predefined agent, define a custom agent with clear scope, model, and type — but this should be rare
- Maximum 6 agents per team. If more are needed, break into phases.

## Communication Matrix

Two modes: **Hub-and-spoke** (default) for implementation work, and **Discussion mode** for debate/investigation tasks. The Orchestrator chooses which mode to use based on the task and offers it to the user.

### Mode A: Hub-and-Spoke (Default)

All communication flows through the Orchestrator. No agent-to-agent direct messaging.

```
                    ┌──────────┐
                    │ Explorer │
                    └────┬─────┘
                         │
┌──────────┐    ┌────────┴────────┐    ┌──────────┐
│ Frontend ├────┤  ORCHESTRATOR   ├────┤ Backend  │
└──────────┘    │    (opus)       │    └──────────┘
                │                 │
┌──────────┐    │  All messages   │    ┌──────────┐
│   iOS    ├────┤  route through  ├────┤ Android  │
└──────────┘    │  this node      │    └──────────┘
                │                 │
┌──────────┐    │                 │    ┌──────────┐
│ Database ├────┤                 ├────┤   API    │
└──────────┘    │                 │    └──────────┘
                │                 │
┌──────────┐    │                 │    ┌──────────┐
│ Debugger ├────┤                 ├────┤ Reviewer │
└──────────┘    └────────┬────────┘    └──────────┘
                         │
                    ┌────┴──────┐
                    │Copywriter │
                    └───────────┘
```

**Use for:** Implementation, feature work, any task where agents write code to distinct files.

#### Hub-and-Spoke Rules

| Rule | Description |
|------|-------------|
| **No cross-talk** | Agents never message each other directly. All goes through Orchestrator. |
| **Results only** | Agents send: what was done, files changed, blockers. No reasoning, no summaries. |
| **Ask when blocked** | If an agent can't proceed, it messages Orchestrator with the specific blocker — not a general status update. |
| **Orchestrator relays context** | If Frontend needs info about the API contract, Orchestrator asks API agent (or Explorer) and relays the answer. |
| **Explorer is Orchestrator's lens** | Orchestrator never searches the codebase itself. It asks Explorer to find what it needs. |
| **Reviewer talks last** | Reviewer only activates after implementation agents complete. Reports to Orchestrator who decides on fixes. |

### Mode B: Discussion Mode (Peer-to-Peer)

Agents message each other directly to debate, challenge, and build on findings. Orchestrator acts as moderator, not relay. **All discussion is logged to a shared file** as a permanent record.

```
┌────────────┐    ┌────────────┐
│ Panelist A ├────┤ Panelist B │
└─────┬──────┘    └──────┬─────┘
      │       ╲  ╱       │
      │        ╲╱        │
      │        ╱╲        │
      │       ╱  ╲       │
┌─────┴──────┐    ┌──────┴─────┐
│ Panelist C ├────┤ Panelist D │
└─────┬──────┘    └──────┬─────┘
      │                  │
      └───────┬──────────┘
         ┌────┴─────┐
         │MODERATOR │
         │ (opus)   │
         └──────────┘
              │
       ┌──────┴──────┐
       │ DISCUSSION  │
       │    LOG      │
       │ (temp/*.md) │
       └─────────────┘
```

**Use for:** Debugging with competing hypotheses, architecture decisions with tradeoffs, design reviews, any task where the goal is consensus through debate rather than parallel execution.

#### Discussion Log Setup

The Moderator creates a shared discussion log file that all panelists write to. This provides a permanent record of the conversation that would otherwise be lost.

**Log location:** `temp/discussions/{team-name}-{timestamp}.md`

**Moderator responsibilities:**
1. Create `temp/discussions/` directory if it doesn't exist
2. Create the log file with header before spawning panelists
3. Include the log file path in each panelist's spawn prompt
4. Append synthesis and consensus to the log at the end

**Log format:**
```markdown
# Discussion Log: {topic}

**Team:** {team-name}
**Started:** {ISO timestamp}
**Participants:** {comma-separated list}
**Question:** {the question being debated}

---

## Round 0: Research

(Panelists append their research findings here)

---

## Round 1: Present

(Panelists append their presentations here)

---

## Round 2: Debate

(Panelists append challenges and responses here)

---

## Consensus

(Moderator appends final synthesis here)
```

**Message signing format:**
```markdown
### [{Agent-Name}] - {Round N} - {HH:MM:SS}

{message content}

---
```

Example:
```markdown
### [Panelist-1] - Round 1 - 14:23:45

Based on my investigation of the authentication flow, I found evidence that
the session timeout is the root cause:

- `auth/session.ts:142` - timeout set to 30 seconds
- `middleware/auth.ts:89` - no session refresh on activity

I believe the bug is in the session timeout configuration.

---
```

**Handling concurrent writes:**
- Panelists must use the **Edit tool** to append, not the Write tool (which overwrites)
- Each panelist appends to their round's section using a unique signature header
- If two panelists write simultaneously, both entries will appear (order may vary)
- The timestamp in the signature helps establish chronological order
- Moderator can reorder entries during synthesis if needed

#### When to Offer Discussion Mode

Detect these signals in the user's request and offer discussion mode as an option:

| Signal | Example |
|--------|---------|
| **Multiple hypotheses** | "Figure out why X is happening" (root cause unknown) |
| **Tradeoff analysis** | "Should we use approach A or B?" |
| **Devil's advocate needed** | "Poke holes in this design" |
| **Consensus-seeking** | "What's the best way to handle X?" |
| **Investigation with unknowns** | "Users report X but we can't reproduce" |

If detected, present both options to the user:

```
This task could use either approach:

1. Hub-and-spoke (standard) — Orchestrator assigns investigation areas,
   agents report back, Orchestrator synthesizes.
2. Discussion mode — Agents investigate independently then debate each
   other's findings. Produces stronger conclusions through adversarial
   testing but uses more tokens (peer messages).

Which approach do you prefer?
```

#### Discussion Mode Rules

| Rule | Description |
|------|-------------|
| **Peer messaging allowed** | Agents message each other directly via `SendMessage` type `message`. |
| **Challenge, don't agree** | Agents must attempt to disprove other agents' conclusions before accepting them. |
| **Evidence required** | Every claim must reference specific files, lines, or logs. No speculation without evidence. |
| **Moderator synthesizes** | Orchestrator monitors the discussion and synthesizes the final consensus. Does not participate in the debate. |
| **Rounds are bounded** | Orchestrator sets a round limit (typically 2-3). After final round, Orchestrator calls for conclusions. |
| **Explorer still serves all** | Explorer remains available to any agent who needs codebase evidence during discussion. |
| **No code changes** | Discussion mode is read-only. If the conclusion requires implementation, switch to hub-and-spoke for the execution phase. |
| **Log all contributions** | Every panelist appends their findings, challenges, and responses to the shared discussion log with their signature. |
| **Moderator owns the log** | Moderator creates the log file, includes its path in spawn prompts, and appends the final consensus. |

#### Discussion Mode Lifecycle

```
1. Orchestrator creates temp/discussions/ folder and initializes log file with header
2. Orchestrator spawns Explorer + 3-5 panelists (Explore type, sonnet)
   - Each panelist's spawn prompt includes the log file path
3. Each panelist investigates independently (Round 0 — research)
   - Panelists append signed research notes to the log
4. Panelists share initial findings with each other (Round 1 — present)
   - Panelists append signed presentations to the log
5. Panelists challenge each other's findings (Round 2 — debate)
   - Panelists append signed challenges and responses to the log
6. Optional: Round 3 if no convergence
7. Orchestrator synthesizes consensus, appends to log, and reports to user
8. If implementation needed: switch to hub-and-spoke with the consensus as spec
```

#### Token Cost of Discussion Mode

Peer messages mean N agents can each send messages to N-1 others. With 4 panelists and 2 debate rounds, that's up to ~24 messages vs ~8 in hub-and-spoke. **Only use when the quality of the conclusion justifies the cost.** For straightforward debugging where the bug is likely in one place, hub-and-spoke is cheaper and sufficient.

## Token Optimization Rules

Baked into every agent's prompt to prevent wasteful behavior.

### All Agents

| Rule | Enforcement |
|------|-------------|
| **Think silently** | All reasoning happens in extended thinking. Output only results, decisions, and blockers. |
| **Scoped search** | Only search directories/files relevant to your task. Never glob `**/*` across the whole project. |
| **Read selectively** | Read specific files, not entire directories. If you need a function, grep for it — don't read the whole file hoping to find it. |
| **No exploratory browsing** | Don't read files "to understand the codebase." You get a file list from the Orchestrator. Work within it. |
| **Expand scope only when blocked** | If your assigned files aren't enough, message Orchestrator explaining what you need and why. Don't search on your own. |
| **Minimal output** | Report format: files changed, what was done, blockers if any. No summaries of what you read, no restating the task. |

### Orchestrator

| Rule | Enforcement |
|------|-------------|
| **Never search directly** | Use Explorer to find files, patterns, conventions. Opus tokens are expensive — don't spend them on grep. |
| **Provide file lists upfront** | Before assigning a task, use Explorer to identify the exact files the agent needs. Include them in the task assignment. |
| **Batch Explorer queries** | Ask Explorer multiple questions in one message rather than one at a time. |

### Explorer

| Rule | Enforcement |
|------|-------------|
| **Targeted searches** | Answer the specific question asked. Don't return extra context "just in case." |
| **Paths over content** | Return file paths and line numbers. Only include code snippets when specifically asked. |
| **Stop when answered** | Once you have the answer, stop searching. Don't keep looking for more matches. |

## Step 1: Analyze the Task

This is the most important step. Extract these dimensions before making any decisions:

### 1a. Identify the Work Units

Break the task into discrete pieces of work. Ask:
- What distinct things need to happen?
- Which pieces can happen in parallel vs must be sequential?
- Do any pieces need results from another piece?

### 1b. Map Work Units to Agents

```
Work unit involves...          → Spawn agent
─────────────────────────────────────────────
Codebase questions             → Explorer (always spawned)
React components/views/styling → Frontend
Rails models/controllers/logic → Backend
API endpoints/serializers      → API
Schema changes/migrations      → Database
Swift/SwiftUI code             → iOS
Kotlin/Compose code            → Android
User-facing text/labels        → Copywriter
Tests/debugging                → Debugger
Quality review                 → Reviewer (always spawned for implementation)
```

### 1c. Common Task-to-Team Mappings

| Task Type | Agents Spawned | Mode |
|-----------|---------------|------|
| Backend bug fix | Explorer + Backend + Debugger + Reviewer | Hub-and-spoke |
| New React feature | Explorer + Frontend + API + Reviewer | Hub-and-spoke |
| Full-stack feature | Explorer + Frontend + Backend + API + Reviewer | Hub-and-spoke |
| Cross-platform feature | Explorer + API + Frontend + iOS + Android + Reviewer | Hub-and-spoke |
| Database migration | Explorer + Database + Backend + Reviewer | Hub-and-spoke |
| UI polish / copy changes | Explorer + Frontend + Copywriter + Reviewer | Hub-and-spoke |
| Investigation / research | Explorer only (multiple queries) | Hub-and-spoke |
| Root cause unknown bug | Explorer + 3-5 panelists | **Offer discussion** |
| Architecture decision | Explorer + 3-4 panelists | **Offer discussion** |
| Design review / critique | Explorer + 3-4 panelists | **Offer discussion** |

### 1d. Produce a Recommendation

Present to the user before executing:

```
Analysis:
- Task breaks into N work units: [list them]
- [X] can run in parallel, [Y] are sequential
- Recommended approach: [single/parallel/team]

Agents:
- [agent] — [why needed]
- [agent] — [why needed]

Execution order:
1. Explorer maps relevant files
2. [task] (parallel with 3)
3. [task] (parallel with 2)
4. Reviewer validates all changes
```

## Step 2: Choose the Approach

### Single Agent
**When:** 1 work unit, or tightly coupled tasks where output of one feeds the next.

Use a single `Task` tool call with the right model. No team overhead.

### Parallel Subagents
**When:** 2+ independent tasks, no shared state, fire-and-forget.

Spawn multiple `Task` calls in a single message. Each runs independently and reports back. No `TeamCreate` needed.

### Full Team
**When any of these apply:**
- 3+ tasks needing coordination or handoffs
- Different specializations required
- Long-running work where agents need to communicate findings
- Cross-layer work (frontend + backend + mobile) requiring coordination

**Do NOT use a team when:**
- A single sonnet agent could do it in one pass
- Tasks are tightly coupled and must run sequentially
- The coordination overhead exceeds the parallelism benefit

## Step 3: Select Models

Default to **sonnet**. The Agent Roster defines default models for each role. Only deviate when justified.

### Model Selection by Task Signal

| Signal | Model | Rationale |
|--------|-------|-----------|
| Search, grep, find files, list patterns | **haiku** | Mechanical, no reasoning |
| Simple edits, boilerplate, formatting | **haiku** | Pattern-following |
| Data gathering across many files | **haiku** | Volume over depth |
| Feature implementation, clear spec | **sonnet** | Strong code gen + reasoning |
| Debugging, root cause analysis | **sonnet** | Needs to trace and hypothesize |
| Code review, test writing | **sonnet** | Judgment within defined scope |
| Architecture design, novel patterns | **opus** | Competing tradeoffs, deep reasoning |
| Ambiguous requirements, open-ended | **opus** | Must make judgment calls |
| Coordinating 4+ agents as lead | **opus** | Orchestration complexity |

### Cost Awareness

- Haiku is ~12x cheaper than Sonnet, ~60x cheaper than Opus
- **Pyramid pattern**: Explorer (haiku) gathers → Implementers (sonnet) build → Orchestrator (opus) coordinates
- Orchestrator's most expensive action is thinking — minimize by delegating all search to Explorer

### Upgrade / Downgrade Signals

**Upgrade when:** >5 files involved, indirect effects to reason about, ambiguous requirements, no existing pattern to follow, mistakes are costly.

**Downgrade when:** similar to existing code, single file, read-only, template-based, repetitive across files.

## Step 4: Compose and Execute

### Lifecycle

1. **Create team** — `TeamCreate` with descriptive name
2. **Spawn Explorer** — always first. Use it to map relevant files.
3. **Create tasks** — `TaskCreate` with subjects, descriptions, file lists, dependencies (`addBlockedBy`)
4. **Spawn agents** — from the roster, only those needed. Include file lists in spawn prompts.
5. **Assign tasks** — `TaskUpdate` with `owner` for each agent
6. **Coordinate** — `SendMessage` type: `message` only. No broadcasts. Relay between agents as needed.
7. **Review** — Reviewer validates all implementation work before completion
8. **Shutdown** — `SendMessage` type: `shutdown_request` to each agent
9. **Cleanup** — `TeamDelete` after all agents have shut down

### Orchestrator Workflow

```
1. Receive task from user
2. Spawn Explorer → ask targeted questions about relevant files
3. Based on Explorer's findings, determine which agents to spawn
4. Create tasks with explicit file lists and scope boundaries
5. Spawn agents, assign tasks
6. Wait for completion. Relay context between agents if needed.
7. Spawn Reviewer once implementation is done
8. If Reviewer finds issues → assign fixes to original agents
9. Report results to user
10. Shutdown all agents → TeamDelete
```

### Team Lead Controls

- **Delegate mode** (Shift+Tab): restricts lead to coordination only — prevents lead from implementing tasks itself.
- **Plan approval**: spawn teammates that must submit plans before implementing. Lead reviews and approves/rejects.
- **Direct interaction**: users can message teammates directly (Shift+Up/Down in-process, click pane in split mode).

## Prompt Template

See [agent-prompt-template.md](agent-prompt-template.md) for role-specific prompt structures.

Key rules:
- Every prompt includes the standard behavior block (silent thinking, minimal output, scoped search)
- Include **file lists** — agents know exactly which files to touch
- Be specific: "Implement the settings modal following the pattern in ProfileModal.tsx" not "add settings"
- Define **scope boundaries** (what NOT to touch)
- Orchestrator prompts include the communication matrix rules

## Constraints and Limitations

- **Permissions inherit** from lead. All teammates start with the lead's permission mode.
- **No nested teams** — teammates cannot spawn their own teams.
- **One team per session** — clean up current team before starting a new one.
- **No session resumption** — `/resume` and `/rewind` don't restore in-process teammates.
- **Lead is fixed** — the session that creates the team is the lead for its lifetime.
- **Task claiming** uses file locking to prevent race conditions.
- **Idle is normal** — teammates go idle after each turn. They wake when messaged.
- **Split-pane mode** requires tmux (not supported in VS Code, Windows Terminal, or Ghostty).

### Background Agent File Writing

Background agents (`run_in_background: true` on `Task`) cannot surface permission prompts. Any tool not pre-approved in `~/.claude/settings.json` is auto-denied with "prompts unavailable."

**The `mode` parameter does NOT fix this.** `bypassPermissions`, `dontAsk`, `acceptEdits` all fail without the allowlist.

**Required settings** for background agents that write files:

```json settings.json
{
  "permissions": {
    "allow": ["Read", "Write", "Edit"]
  }
}
```

**Bash** is intentionally omitted — pre-approving it lets agents run any shell command (including destructive ones). Only add `"Bash"` if you accept that risk.

**Alternative pattern** if you don't want to pre-approve tools: use Explore-type agents for research, then write files from the orchestrator/main session.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Agents thinking out loud | Enforce "all reasoning in extended thinking" in every prompt |
| Orchestrator searching the codebase directly | Always delegate to Explorer — saves opus tokens |
| Agents browsing beyond their scope | Include explicit file lists and "do not search beyond" in prompts |
| Using opus for everything | Only Orchestrator is opus. Everyone else is sonnet/haiku. |
| Creating team for single task | Use single `Task` tool call instead |
| No task dependencies | Use `addBlockedBy` to prevent race conditions |
| Broadcasting when DM works | Use `message` type only — no broadcasts |
| Two teammates editing same file | Assign distinct file ownership via the roster scopes |
| general-purpose for read-only work | Use Explore type (Explorer agent) |
| Forgetting to shut down agents | Always send `shutdown_request` when done |
| TeamDelete with active agents | Must shutdown all agents first |
| No review stage | Reviewer is always present for implementation work |
| Agents lack context | Include full task details and file lists in spawn prompt |
| Agents restating their task | Enforce "no restating, no summaries" in behavior block |
| Background agents can't write files | Add `Read`, `Write`, `Edit` to `permissions.allow` in settings.json |
| Discussion mode without log file | Moderator must create log before spawning panelists |
| Panelists not signing messages | Every log entry needs `### [Agent-Name] - Round N - HH:MM:SS` header |
| Log writes overwriting each other | Panelists use Edit tool to append, not Write tool to overwrite |
| Discussion log lost | Always report the log file path to the user at the end |

## Red Flags

**Overbuilding:** team has >6 agents, multiple opus agents, circular dependencies, more time coordinating than working.

**Underbuilding:** single agent doing 5+ independent things sequentially, clearly parallelizable work running serially, opus doing file searches.

**Token waste:** agents outputting reasoning text, Orchestrator running grep/glob itself, agents reading entire directories, Explorer returning full file contents instead of paths.

## Reference

See [research.md](research.md) for detailed model comparisons, agent type capabilities, permission modes, communication patterns, real-world configurations, and anti-patterns.
