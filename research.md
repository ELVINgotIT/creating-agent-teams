# Research: Claude Code Agent Teams

> Reference material for the "creating-agent-teams" skill. Covers model selection, team composition, workflow patterns, and decision frameworks.

**Official documentation:** [Agent Teams on code.claude.com](https://code.claude.com/docs/en/agent-teams)

---

## 1. Model Capabilities and Cost/Speed Tradeoffs

The `model` parameter on agents and Task tool accepts: `"haiku"`, `"sonnet"`, `"opus"`.

### Model Comparison Matrix

| Dimension | Haiku | Sonnet | Opus |
|-----------|-------|--------|------|
| **Speed** | Fastest | Moderate | Slowest |
| **Cost** | Lowest (~$0.25/$1.25 per M tokens) | Mid (~$3/$15 per M tokens) | Highest (~$15/$75 per M tokens) |
| **Reasoning** | Basic | Strong | Deepest |
| **Context Handling** | Good for focused tasks | Good for moderate complexity | Best for multi-file, ambiguous tasks |
| **Code Generation** | Boilerplate, simple edits | Feature implementation, debugging | Architecture, complex refactoring |
| **When to use** | High-volume, simple tasks | Default workhorse | Critical decisions, skill/prompt writing |

### Haiku: Best For

- Simple file searches and glob/grep operations
- Code formatting and linting fixes
- Boilerplate generation (tests, docs from templates)
- Straightforward single-file edits
- Data extraction and summarization
- Running predefined scripts and reporting results
- Quick validation checks (file exists, pattern matches)

### Sonnet: Best For (DEFAULT CHOICE)

- Feature implementation across 1-5 files
- Code review with confidence scoring
- Debugging and root cause analysis
- Test writing with edge cases
- Moderate complexity reasoning
- Codebase exploration and architecture mapping
- Most agent roles in a team (explorer, implementer, reviewer)

### Opus: Best For

- Complex architecture decisions spanning many files
- Multi-file refactoring with subtle interdependencies
- Nuanced design problems with competing tradeoffs
- Writing skills, prompts, and agent system prompts (meta-work)
- Tasks requiring deep understanding of unfamiliar codebases
- Resolving ambiguous requirements
- Team leadership / orchestration of complex multi-agent workflows
- Anything where a wrong decision is very costly to undo

### Cost Multipliers (Approximate)

- Haiku is ~12x cheaper than Sonnet per token
- Sonnet is ~5x cheaper than Opus per token
- Haiku is ~60x cheaper than Opus per token

**Practical implication**: A team of 5 Haiku agents costs roughly the same as 1 Sonnet agent. A team of 5 Sonnet agents costs roughly the same as 1 Opus agent. This means you can massively parallelize with cheaper models.

---

## 2. Agent Types (subagent_type parameter)

The `subagent_type` parameter on the Task tool determines what tools the spawned agent has access to.

### Agent Type Capabilities

| Type | Tools Available | Can Edit Files? | Can Spawn Sub-tasks? | Best For |
|------|----------------|-----------------|---------------------|----------|
| **Bash** | Bash only | No (only via bash) | No | Git operations, terminal commands, process management |
| **general-purpose** | All tools (Read, Write, Edit, Bash, Glob, Grep, Task, etc.) | Yes | Yes (can spawn sub-tasks) | Full implementation, multi-step workflows |
| **Explore** | Read, Glob, Grep, Bash (read-only intent), WebFetch, WebSearch | No | No | Codebase exploration, research, file discovery |
| **Plan** | Read, Glob, Grep, Bash (read-only intent), WebFetch, WebSearch | No | No | Architecture planning, design documents |
| **superpowers:code-reviewer** | All tools | Yes | Yes | Code review with ability to suggest and apply fixes |

### Detailed Type Descriptions

#### Bash Agent
- **Only tool**: Bash
- **Use when**: You need git operations, system commands, process management, or CLI tool execution
- **Example tasks**: `git log`, `npm install`, `docker build`, running test suites, checking system state
- **Limitations**: Cannot use Read/Write/Edit tools directly; must use bash equivalents (cat, sed, etc.)

#### General-Purpose Agent
- **All tools available**: Read, Write, Edit, Bash, Glob, Grep, Task, ExitPlanMode, NotebookEdit, etc.
- **Use when**: The agent needs to make code changes, create files, or coordinate sub-tasks
- **Example tasks**: Implementing features, fixing bugs, creating new files, multi-step workflows
- **Key advantage**: Can spawn its own sub-tasks (recursive delegation)

#### Explore Agent
- **Tools**: Read, Glob, Grep, Bash (read-only), WebFetch, WebSearch
- **Explicitly excluded**: Task, ExitPlanMode, Edit, Write, NotebookEdit
- **Use when**: You need codebase understanding without risk of modifications
- **Example tasks**: "Find all files related to authentication", "Map the data flow for payments", "List all API endpoints"
- **Key advantage**: Safe - cannot modify anything. Perfect for parallel research.

#### Plan Agent
- **Tools**: Same as Explore (Read, Glob, Grep, Bash read-only, WebFetch, WebSearch)
- **Explicitly excluded**: Task, ExitPlanMode, Edit, Write, NotebookEdit
- **Use when**: You need architecture design or implementation planning
- **Example tasks**: "Design the architecture for a new feature", "Create an implementation blueprint"
- **Distinction from Explore**: Semantic - Plan agents are expected to produce structured plans/blueprints, while Explore agents produce research/findings. The tool access is identical.

#### superpowers:code-reviewer
- **All tools available** (same as general-purpose)
- **Use when**: You need code review that can reference project standards and potentially fix issues
- **Key advantage**: Specialized for review workflows with ability to apply fixes

### Agent Type Selection Heuristic

```
Does the agent need to modify files?
  YES → Does it need to spawn sub-tasks?
    YES → general-purpose
    NO  → general-purpose (still the right choice)
  NO  → Is it primarily researching/exploring?
    YES → Explore
    NO  → Is it designing/planning?
      YES → Plan
      NO  → Is it running shell commands only?
        YES → Bash
        NO  → Explore (safe default for read-only)
```

---

## 3. Permission Modes

Permission modes control what the agent can do without asking the user for approval.

| Mode | Description | When to Use |
|------|-------------|-------------|
| `default` | Standard permission checks; user approves risky actions | Default for most agents |
| `acceptEdits` | Auto-approves file edits (Read, Write, Edit) but still prompts for Bash | Trusted implementation agents |
| `bypassPermissions` | No permission prompts at all | Fully trusted autonomous agents (use with caution) |
| `plan` | Read-only mode; cannot make changes | Planning/exploration agents |
| `delegate` | Can delegate to sub-agents but limited direct action | Orchestrator/coordinator agents |
| `dontAsk` | Never asks user; fails silently if permission denied | Background/automated agents |

### Permission Mode Selection

- **Explore/Plan agents**: `plan` mode is natural since they are read-only
- **Implementation agents**: `acceptEdits` is practical since the whole point is making changes
- **Review agents**: `default` is good - they mostly read, occasionally suggest
- **Autonomous agents**: `bypassPermissions` only when you truly trust the agent and want zero interruption
- **Team leads**: `default` or `delegate` depending on whether they implement or only orchestrate

---

## 4. Team Workflow Patterns

### Team Lifecycle

```
1. CREATE TEAM (TeamCreate)
   └── Define team name, roles, and agent configurations

2. CREATE TASKS (TaskCreate)
   └── Define work items with subjects, descriptions, dependencies
   └── Set up blockedBy/blocks relationships

3. SPAWN AGENTS (agents start working)
   └── Each agent claims tasks and marks them in_progress
   └── Agents use tools to complete work

4. COMMUNICATE (SendMessage)
   └── Agents message each other for coordination
   └── Team lead broadcasts critical updates
   └── Agents report findings and request help

5. REVIEW (code-reviewer agents)
   └── Review agents check completed work
   └── Issues reported back via messages/tasks

6. SHUTDOWN (shutdown_request/response)
   └── Leader sends shutdown requests when work is complete
   └── Agents approve or reject (if still working)
   └── Clean termination

7. DELETE TEAM
   └── Team resources cleaned up
```

### Task Management (TaskCreate, TaskList, TaskUpdate)

**TaskCreate** fields:
- `subject`: Brief imperative title ("Fix authentication bug in login flow")
- `description`: Detailed requirements, context, acceptance criteria
- `activeForm`: Present continuous shown in spinner ("Fixing authentication bug")

**TaskUpdate** operations:
- Set status: `pending` -> `in_progress` -> `completed`
- Set owner (claim task)
- Add dependencies: `addBlocks`, `addBlockedBy`
- Delete tasks: `status: "deleted"`

**TaskList** usage:
- Check available work (pending, no owner, not blocked)
- Monitor overall progress
- Find newly unblocked tasks after completing dependencies

**Task dependency pattern**:
```
Task 1: Research (no dependencies)
Task 2: Design (blockedBy: [1])
Task 3: Implement (blockedBy: [2])
Task 4: Review (blockedBy: [3])
```

### Communication (SendMessage)

**Message types**:

| Type | Use Case | Cost |
|------|----------|------|
| `message` | Direct message to one teammate | Low (1 delivery) |
| `broadcast` | Message to ALL teammates | High (N deliveries) |
| `shutdown_request` | Ask teammate to shut down | Low |
| `shutdown_response` | Approve/reject shutdown | Low |
| `plan_approval_response` | Approve/reject a teammate's plan | Low |

**Critical rules**:
- Plain text output is NOT visible to teammates - MUST use SendMessage
- Use `message` (DM) as default - `broadcast` is expensive
- Broadcast only for critical team-wide announcements
- Always refer to teammates by NAME, not UUID

### Idle/Wake Pattern

- Agents go idle when they have no more tasks to work on
- Idle agents are notified when new tasks appear or messages arrive
- Agents should call TaskList after completing work to find next task
- Prefer tasks in ID order (lowest first) when multiple are available

---

## 5. Common Team Compositions

### Pattern A: Solo Agent + Reviewer

**Structure**: 1 implementer + 1 code reviewer
**Best for**: Single features, bug fixes, moderate changes

```
Team:
  - implementer (general-purpose, sonnet)
    Role: Implement the feature/fix
  - reviewer (superpowers:code-reviewer, sonnet)
    Role: Review completed work

Task flow:
  1. Implement feature (implementer)
  2. Review implementation (reviewer, blockedBy: [1])
```

**Pros**: Simple, effective quality gate
**Cons**: Sequential - reviewer waits for implementer

### Pattern B: Research Team (Parallel Explorers)

**Structure**: 2-4 explore agents working in parallel
**Best for**: Codebase exploration, understanding unfamiliar code, research

```
Team:
  - explorer-1 (Explore, sonnet) → "Trace authentication flow"
  - explorer-2 (Explore, sonnet) → "Map database schema and relationships"
  - explorer-3 (Explore, sonnet) → "Find all API endpoints and middleware"

Task flow: All tasks run in parallel (no dependencies)
```

**Pros**: Fast parallel exploration, safe (read-only), cost-efficient
**Cons**: May overlap in findings; need synthesis step

**Real example**: The `feature-dev` plugin launches 2-3 `code-explorer` agents in parallel during Phase 2 (Codebase Exploration), each targeting a different aspect.

### Pattern C: Implementation Team

**Structure**: Leader + implementers + reviewer
**Best for**: Large features, multi-file changes

```
Team:
  - team-lead (general-purpose, opus)
    Role: Coordinate, make architecture decisions, synthesize
  - frontend-dev (general-purpose, sonnet)
    Role: Implement frontend changes
  - backend-dev (general-purpose, sonnet)
    Role: Implement backend changes
  - reviewer (superpowers:code-reviewer, sonnet)
    Role: Review all completed work

Task flow:
  1. Research codebase (parallel explorers, or lead does this)
  2. Design architecture (lead or Plan agent)
  3a. Implement frontend (frontend-dev, blockedBy: [2])
  3b. Implement backend (backend-dev, blockedBy: [2])
  4. Review all (reviewer, blockedBy: [3a, 3b])
```

**Pros**: Parallel implementation, specialized agents, quality gate
**Cons**: More expensive, coordination overhead

### Pattern D: Feature Development (feature-dev pattern)

**Structure**: Multi-phase with different agent types per phase
**Best for**: End-to-end feature development with quality

```
Phase 1: Discovery (lead asks clarifying questions)
Phase 2: Exploration (2-3 code-explorer agents, sonnet)
Phase 3: Clarifying Questions (lead synthesizes, asks user)
Phase 4: Architecture (2-3 code-architect agents, sonnet)
Phase 5: Implementation (lead implements chosen approach)
Phase 6: Quality Review (3 code-reviewer agents, sonnet)
Phase 7: Summary (lead documents)
```

**Key insight**: This pattern uses different agent TYPES at different phases. Explorers first (read-only, safe), then architects (read-only, planning), then implementation (write), then reviewers.

### Pattern E: PR Review Team (Parallel Reviewers)

**Structure**: Multiple specialized reviewers in parallel
**Best for**: Comprehensive code review

```
Team:
  - comment-analyzer (Explore, sonnet) → Check comment accuracy
  - test-analyzer (Explore, sonnet) → Review test coverage
  - silent-failure-hunter (Explore, sonnet) → Find silent failures
  - type-design-analyzer (Explore, sonnet) → Analyze type design
  - code-reviewer (Explore, sonnet) → General code quality

Task flow: All run in parallel on the same diff
```

**Pros**: Deep specialized review, fast (parallel), comprehensive
**Cons**: Multiple agents = higher cost

### Pattern F: Research + Synthesis

**Structure**: Multiple researchers + one synthesizer
**Best for**: Complex investigation, learning new codebases

```
Team:
  - researcher-1 (Explore, haiku) → Gather data on topic A
  - researcher-2 (Explore, haiku) → Gather data on topic B
  - researcher-3 (Explore, haiku) → Gather data on topic C
  - synthesizer (general-purpose, opus) → Combine findings

Task flow:
  1a-c. Research (parallel, haiku for speed/cost)
  2. Synthesize (opus for deep reasoning, blockedBy: [1a, 1b, 1c])
```

**Key insight**: Use cheap fast models for data gathering, expensive smart model for synthesis. This is one of the most cost-effective patterns.

---

## 6. Model Selection Heuristics

### Decision Framework

```
Step 1: Assess Task Complexity
  - Simple (grep, format, boilerplate) → Haiku
  - Moderate (implement, debug, review) → Sonnet
  - Complex (architecture, ambiguous requirements, meta-work) → Opus

Step 2: Assess Task Count
  - Many similar tasks (5+) → Use cheaper model, parallelize
  - Few unique tasks (1-3) → Use smarter model for quality

Step 3: Assess Risk
  - Low risk (read-only, exploration) → Haiku or Sonnet
  - Medium risk (implementation with tests) → Sonnet
  - High risk (architecture, irreversible decisions) → Opus

Step 4: Assess Volume
  - High volume, repeated pattern → Haiku
  - Standard development → Sonnet
  - One-time critical decision → Opus
```

### Model Selection by Role

| Team Role | Recommended Model | Rationale |
|-----------|-------------------|-----------|
| Team Lead / Orchestrator | Opus | Needs deep reasoning for coordination and decisions |
| Codebase Explorer | Sonnet | Needs good understanding to trace code flows |
| Architecture Designer | Sonnet or Opus | Depends on complexity; Opus for novel architectures |
| Feature Implementer | Sonnet | Strong code generation with good reasoning |
| Test Writer | Sonnet | Needs to understand code AND think about edge cases |
| Code Reviewer | Sonnet | Needs nuanced judgment about code quality |
| Simple File Searcher | Haiku | Just running grep/glob and reporting results |
| Boilerplate Generator | Haiku | Pattern-following, not creative reasoning |
| Documentation Writer | Sonnet | Needs to understand code to document it well |
| Bug Fixer (simple) | Sonnet | Needs debugging skills |
| Bug Fixer (complex) | Opus | Subtle bugs need deep reasoning |
| Skill/Prompt Writer | Opus | Meta-cognitive task requiring best reasoning |
| Data Gatherer | Haiku | Collecting information, not analyzing |
| Synthesizer | Opus | Combining and reasoning over multiple inputs |

### When to Upgrade Models

Signals that you should use a smarter model:

1. **Number of files**: >5 files involved → consider Opus
2. **Reasoning depth**: Requires understanding indirect effects → Opus
3. **Ambiguity**: Requirements are unclear or conflicting → Opus
4. **Novelty**: No existing pattern to follow → Opus
5. **Irreversibility**: Mistakes are expensive to fix → Opus
6. **Interdependency**: Changes in one file affect many others → Opus

Signals that you can use a cheaper model:

1. **Pattern following**: Similar to existing code → Haiku
2. **Single file**: Changes isolated to one file → Haiku/Sonnet
3. **Read-only**: Just gathering information → Haiku
4. **Template-based**: Following a known template → Haiku
5. **Repetitive**: Same task across many files → Haiku

### Cost Optimization Strategies

1. **Pyramid pattern**: Many Haiku gatherers -> Few Sonnet processors -> One Opus synthesizer
2. **Explorers are cheap**: Use Explore agents (read-only) for research, reserve general-purpose for implementation
3. **Phase-based model selection**: Start with cheaper models for exploration, escalate for implementation
4. **Parallel cheap > Sequential expensive**: 5 parallel Haiku tasks often beat 1 sequential Opus task for information gathering
5. **Singleton Opus for decisions**: Only 1 Opus agent for critical decisions; everyone else is Sonnet/Haiku

---

## 7. Agent System Prompt Patterns

### Proven Structure (from agent-creation-system-prompt.md)

```markdown
You are [specific role] specializing in [specific domain].

**Your Core Responsibilities:**
1. [Primary responsibility]
2. [Secondary responsibility]
3. [Additional as needed]

**[Task Name] Process:**
1. [Step 1]
2. [Step 2]
...

**Quality Standards:**
- [Standard 1]
- [Standard 2]

**Output Format:**
[Specify exact format]

**Edge Cases:**
- [Case 1]: [Handling]
- [Case 2]: [Handling]
```

### Key Principles for Agent Prompts

1. **Be specific, not vague** - "Check for SQL injection by examining parameterized queries" NOT "look for security issues"
2. **Use second person** - "You are...", "You will..."
3. **Include concrete steps** - Give the agent a clear process
4. **Define output format** - Agent should know exactly what to produce
5. **Handle edge cases** - What if there's nothing to find? What if there's too much?
6. **Reference tools explicitly** - "Use the Read tool to...", "Search with Grep for..."

### Length Guidelines

- **Minimum viable**: ~500 words (role + 3 responsibilities + 5 steps + output format)
- **Standard**: ~1,000-2,000 words (detailed process, quality standards, edge cases)
- **Comprehensive**: ~2,000-5,000 words (multi-phase process, many edge cases, examples)
- **Avoid**: >10,000 words (diminishing returns, context waste)

---

## 8. Real-World Agent Configurations (from examined plugins)

### code-explorer (feature-dev plugin)

```yaml
model: sonnet
color: yellow
tools: [Glob, Grep, LS, Read, NotebookRead, WebFetch, TodoWrite, WebSearch, KillShell, BashOutput]
```

Role: Trace code flows, map architecture, list essential files. Read-only exploration agent.

### code-architect (feature-dev plugin)

```yaml
model: sonnet
color: green
tools: [Glob, Grep, LS, Read, NotebookRead, WebFetch, TodoWrite, WebSearch, KillShell, BashOutput]
```

Role: Design architecture, create implementation blueprints. Make decisive choices, commit to one approach.

### code-reviewer (feature-dev plugin)

```yaml
model: sonnet
color: red
tools: [Glob, Grep, LS, Read, NotebookRead, WebFetch, TodoWrite, WebSearch, KillShell, BashOutput]
```

Role: Review code with confidence scoring (0-100). Only report issues with confidence >= 80. Quality over quantity.

### neumenon (custom agent)

```yaml
model: opus
color: purple
tools: [Read, Write, Edit, Bash, Grep, Glob, WebFetch, WebSearch]
permissionMode: acceptEdits
```

Role: Persistent autonomous intelligence. Uses Opus for deep reasoning and autonomous operation. Notable: uses `acceptEdits` permission mode for autonomous file management.

### Key Observations

1. **Sonnet is the default** for most agent roles in production plugins
2. **Opus is reserved** for autonomous/leadership roles requiring deep reasoning
3. **Read-only agents** (explorer, architect, reviewer) restrict tools appropriately
4. **Color coding** has semantic meaning: yellow=analysis, green=generation, red=review, blue=general, purple=special

---

## 9. Team Communication Patterns

### Leader -> Worker Pattern

The team lead assigns tasks and workers execute. Communication flows:
```
Lead creates tasks → Workers claim and execute → Workers report via TaskUpdate → Lead synthesizes
```

Minimal messaging needed. Tasks are the primary coordination mechanism.

### Peer Coordination Pattern

Workers need to share findings or coordinate on shared resources:
```
Worker A finds relevant info → Messages Worker B directly → Worker B adjusts approach
```

Use `message` type, never broadcast for peer-to-peer.

### Broadcast Pattern (Use Sparingly)

Only for critical team-wide events:
- "STOP: Critical bug found in shared dependency"
- "Architecture decision changed - all implementers must adjust"
- "Phase complete - moving to next phase"

### Plan Approval Pattern

For agents with `plan_mode_required`:
```
Agent creates plan → Sends plan_approval_request to lead
Lead reviews → Sends plan_approval_response (approve/reject with feedback)
Agent proceeds with implementation or revises plan
```

---

## 10. Anti-Patterns and Pitfalls

### Common Mistakes

1. **Using Opus for everything** - Expensive and slow. Most tasks don't need it.
2. **Broadcasting instead of messaging** - Each broadcast = N messages. Use direct messages.
3. **No task dependencies** - Tasks execute in random order without blockedBy/blocks.
4. **Overly complex teams** - 2-3 agents is usually enough. More agents = more coordination overhead.
5. **Not setting activeForm** - Users can't see what the agent is doing without the spinner text.
6. **Agents duplicating work** - Give clear, non-overlapping scope to each agent.
7. **Sequential when parallel is possible** - Independent research tasks should run in parallel.
8. **Using general-purpose for read-only tasks** - Use Explore type to prevent accidental modifications.
9. **Not specifying output format** - Agents produce inconsistent, hard-to-synthesize results.
10. **Forgetting to shut down agents** - Idle agents still consume resources.

### Team Size Guidelines

| Project Size | Recommended Team Size | Composition |
|-------------|----------------------|-------------|
| Small fix | 1-2 agents | 1 implementer + optional reviewer |
| Medium feature | 2-4 agents | Explorers + implementer + reviewer |
| Large feature | 3-6 agents | Lead + explorers + implementers + reviewer |
| Full project | 4-8 agents | Lead + specialists + reviewers |

**Rule of thumb**: If you need >6 agents, consider breaking the work into phases instead.

---

## 11. Skill Integration Notes

### How This Maps to a Skill

The "creating-agent-teams" skill should:

1. **Trigger on**: User asks to "create a team", "set up agents", "parallelize work", "build a team", or mentions "agent team", "multi-agent"
2. **Provide**: Decision framework for model selection, team composition templates, task dependency patterns
3. **Reference files**: Detailed examples and templates for common team patterns
4. **Key workflow**: Analyze task -> Select team pattern -> Choose models -> Define agents -> Create tasks -> Execute

### SKILL.md Structure Recommendation

```
creating-agent-teams/
├── SKILL.md                        # Core workflow and decision framework
└── references/
    ├── model-selection.md          # Detailed model comparison and heuristics
    ├── team-patterns.md            # Common team compositions with examples
    └── agent-prompt-templates.md   # System prompt templates for common roles
```

Keep SKILL.md lean (~2000-3000 words) with the core decision framework and workflow. Move detailed reference material to references/ files that Claude loads as needed.

---

## 12. Official Documentation Details (from code.claude.com/docs/en/agent-teams)

### Prerequisites

Agent teams are experimental and disabled by default. Must be enabled via environment variable:

```json settings.json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

### Architecture

| Component | Role |
|-----------|------|
| **Team lead** | Main Claude Code session that creates the team, spawns teammates, coordinates work |
| **Teammates** | Separate Claude Code instances that each work on assigned tasks |
| **Task list** | Shared list of work items that teammates claim and complete |
| **Mailbox** | Messaging system for communication between agents |

Storage locations:
- Team config: `~/.claude/teams/{team-name}/config.json`
- Task list: `~/.claude/tasks/{team-name}/`

### Display Modes

| Mode | Description | How to interact |
|------|-------------|-----------------|
| **in-process** (default) | All teammates run inside main terminal | Shift+Up/Down to select teammate, type to message. Enter to view session, Escape to interrupt. Ctrl+T to toggle task list. |
| **split-pane** | Each teammate gets its own pane | Click into pane to interact directly. Requires tmux. |
| **auto** (default setting) | Uses split panes if already in tmux, otherwise in-process | Automatic |

Configure via settings.json:
```json
{
  "teammateMode": "in-process"
}
```

Or per-session: `claude --teammate-mode in-process`

Split-pane requirements: tmux. NOT supported in VS Code integrated terminal, Windows Terminal, or Ghostty.

### Delegate Mode

Press **Shift+Tab** to cycle into delegate mode. Restricts the lead to coordination-only tools: spawning, messaging, shutting down teammates, and managing tasks. Prevents lead from implementing tasks itself.

Use when: the lead should focus entirely on orchestration — breaking down work, assigning tasks, and synthesizing results — without touching code directly.

### Plan Approval

Spawn teammates that must plan before implementing. The teammate works in read-only plan mode until the lead approves:

1. Teammate creates plan
2. Teammate sends `plan_approval_request` to lead
3. Lead reviews and sends `plan_approval_response` (approve or reject with feedback)
4. If rejected, teammate revises and resubmits
5. If approved, teammate exits plan mode and implements

Lead makes approval decisions autonomously. Influence via prompt: "only approve plans that include test coverage" or "reject plans that modify the database schema."

### Permissions

- All teammates start with the lead's permission settings
- If lead runs with `--dangerously-skip-permissions`, all teammates do too
- Can change individual teammate modes after spawning
- Cannot set per-teammate modes at spawn time

### Context Inheritance

Each teammate has its own context window. When spawned:
- Loads same project context as a regular session (CLAUDE.md, MCP servers, skills)
- Receives the spawn prompt from the lead
- Does NOT inherit the lead's conversation history

### Task Claiming

Uses file locking to prevent race conditions when multiple teammates try to claim the same task simultaneously.

### Subagents vs Agent Teams

| Aspect | Subagents | Agent Teams |
|--------|-----------|-------------|
| **Context** | Own context; results return to caller | Own context; fully independent |
| **Communication** | Report results back to main agent only | Teammates message each other directly |
| **Coordination** | Main agent manages all work | Shared task list with self-coordination |
| **Best for** | Focused tasks where only the result matters | Complex work requiring discussion and collaboration |
| **Token cost** | Lower: results summarized back | Higher: each is a separate Claude instance |

Use subagents when: quick focused workers that report back.
Use agent teams when: teammates need to share findings, challenge each other, coordinate on their own.

### Best Practices (from official docs)

1. **Give teammates enough context** — include task-specific details in spawn prompt since they don't inherit conversation history
2. **Size tasks appropriately** — not too small (overhead > benefit), not too large (risk of wasted effort). Self-contained units with clear deliverables.
3. **5-6 tasks per teammate** keeps everyone productive and lets lead reassign if someone gets stuck
4. **Wait for teammates** — lead sometimes starts implementing instead of waiting. Use delegate mode or tell it to wait.
5. **Start with research and review** — clear boundaries, no code writing, shows value of parallel exploration
6. **Avoid file conflicts** — break work so each teammate owns different files. Two teammates editing same file → overwrites.
7. **Monitor and steer** — check progress, redirect failing approaches, synthesize findings as they come in

### Use Case Examples (from official docs)

**Parallel code review:**
```
Create an agent team to review PR #142. Spawn three reviewers:
- One focused on security implications
- One checking performance impact
- One validating test coverage
Have them each review and report findings.
```
Each reviewer applies a different filter. Lead synthesizes across all three.

**Competing hypotheses debugging:**
```
Users report the app exits after one message instead of staying connected.
Spawn 5 agent teammates to investigate different hypotheses. Have them talk to
each other to try to disprove each other's theories, like a scientific
debate. Update the findings doc with whatever consensus emerges.
```
The debate structure prevents anchoring bias. Multiple investigators actively trying to disprove each other — the surviving theory is more likely correct.

### Limitations (from official docs)

1. **No session resumption** — `/resume` and `/rewind` don't restore in-process teammates. Lead may try to message non-existent teammates after resume.
2. **Task status can lag** — teammates sometimes fail to mark tasks completed, blocking dependents. May need manual status update.
3. **Shutdown can be slow** — teammates finish current request/tool call before shutting down.
4. **One team per session** — clean up current team before starting a new one.
5. **No nested teams** — teammates cannot spawn their own teams or teammates. Only the lead manages the team.
6. **Lead is fixed** — the session that creates the team is the lead for its lifetime. Cannot promote or transfer leadership.
7. **Permissions set at spawn** — all teammates start with lead's mode. Can change after, not at spawn time.
8. **Split panes require tmux** — not supported in VS Code, Windows Terminal, or Ghostty.

### Troubleshooting

- **Teammates not appearing**: In in-process mode, press Shift+Down to cycle. Check if task was complex enough for Claude to warrant a team. For split panes, verify tmux is in PATH.
- **Too many permission prompts**: Pre-approve common operations in permission settings before spawning.
- **Teammates stopping on errors**: Check output via Shift+Up/Down, give additional instructions, or spawn replacement.
- **Lead shuts down early**: Tell it to keep going or wait for teammates.
- **Orphaned tmux sessions**: `tmux ls` then `tmux kill-session -t <session-name>`.

---

## 13. Summary of Key Findings

1. **Model selection is the highest-leverage decision** - wrong model = wasted money or poor quality
2. **Sonnet is the default** - use it unless you have a specific reason for Haiku or Opus
3. **Explore agents are the safest parallel pattern** - read-only, cheap, fast
4. **Task dependencies are critical** - without them, agents race and produce conflicts
5. **The pyramid pattern (cheap gather -> smart synthesize) is most cost-effective**
6. **Team size should be 2-6 agents** for most use cases
7. **Communication should be task-driven, not message-driven** - use TaskUpdate over SendMessage
8. **Agent prompts need structure** - role, responsibilities, process, output format, edge cases
9. **Permission modes should match agent roles** - explorers get plan mode, implementers get acceptEdits
10. **Phase-based workflows** (explore -> design -> implement -> review) consistently produce the best results
