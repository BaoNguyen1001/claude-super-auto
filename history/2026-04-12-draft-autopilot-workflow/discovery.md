# Discovery Report — Claude Super Auto

## Project Context

Greenfield project. Empty repo. Building a local-scope Claude Code configuration that provides an autonomous idea-to-implementation workflow loop.

## Existing Ecosystem Assets

### Configuration Primitives

| Type | Location | Format | Key Fields |
|------|----------|--------|------------|
| Agent | `~/.claude/agents/*.md` | Markdown + YAML frontmatter | `name`, `description`, `tools` (optional), `model` (optional) |
| Skill | `~/.claude/skills/<name>/SKILL.md` | Markdown + YAML frontmatter | `name`, `description`, `model` (optional) |
| Command | `~/.claude/commands/*.md` | Plain markdown, no frontmatter | Numbered steps, procedural |

### Reusable Skills

| Skill | What It Does | How We'd Use It |
|-------|-------------|-----------------|
| `brainstorming` | Pre-implementation design gathering, asks questions, writes design doc | Pattern for discussion phase structure |
| `planning` | 8-phase pipeline: discovery → decomposition → track planning | Invoke directly for planning phase |
| `orchestrator` | Spawns parallel workers, Agent Mail coordination, verification | Invoke directly for build phase |
| `file-beads` | Creates bead files from a plan | Used by planning skill |
| `review-beads` | Optimizes bead quality before execution | Used by planning skill |
| `recipe` | Chains commands in sequence (new-feature, bug-fix, etc.) | Pattern for our phase chaining |
| `session-state` | `.ccu/` directory management, crash recovery | Pattern for `.autopilot/` state |

### Reusable Agents

| Agent | Purpose | Reuse |
|-------|---------|-------|
| `worker` | TDD-based bead implementation | Direct — build phase workers |
| `architect` | System design (read-only, opus) | Could be discussion participant |
| `code-reviewer` | Post-implementation quality | Post-build verification |
| `oracle` | Advisory reasoning (read-only) | Evaluation/convergence scoring |

### Coordination Infrastructure

- **Agent Mail (MCP)**: Thread-based async messaging — `register_agent()`, `send_message()`, `fetch_inbox()`, `summarize_thread()`
- **Beads (`br`)**: Task/epic tracking with status, dependencies, file scopes
- **Beads Viewer (`bv`)**: Dependency graph analysis, parallel track planning (`--robot-plan`, `--robot-triage`)
- **Git**: Verification layer — orchestrator inspects diffs for quality smells

### Existing Loop/Autonomous Patterns

1. **Continuous-Agent-Loop** (v1.8+ in everything-claude-code plugin):
   - 3 modes: `continuous-pr`, `rfc-dag`, `sequential`
   - Closest existing pattern to what we're building

2. **Loop-Operator Agent** (kiro/agents):
   - Monitors progress via checkpoints, detects stalls
   - Enforces quality gates, eval baselines, rollback paths
   - Escalates on repeated failures, cost drift

3. **Recipe Skill**:
   - Chains: discuss → plan → review-beads → execute → quality → commit
   - Skips completed phases via `.ccu/CHECKPOINT.md`

4. **`/loop` runtime feature**:
   - Interval-based execution: `/loop 5m /foo`
   - Could drive the outer autopilot loop

### Hooks System

Lifecycle triggers available:
- `PreToolUse` — before Bash/Edit/Write
- `PostToolUse` — after tool completion
- `Stop` — session end
- `SessionStart/SessionEnd` — lifecycle markers

### State Management Patterns

From `session-state` skill:
- **Ephemeral** (gitignored): SESSION.md, CHECKPOINT.md, CAPTURES.md, HANDOFF.md
- **Persistent** (committed): EVIDENCE.md, DECISIONS.md, REQUIREMENTS.md
- Append-only pattern prevents data loss
- Checkpoints enable crash recovery

## Key Gaps (What We Need to Build)

1. **Autopilot Controller** — the meta-loop that chains discuss → plan → build → evaluate → repeat
2. **Discussion Agents** — adversarial Proposer/Challenger pair (no existing pattern)
3. **Evaluator Agent** — goal satisfaction + diminishing returns scoring (no existing pattern)
4. **`.autopilot/` State Schema** — iteration tracking, convergence scores, discussion transcripts
5. **`/autopilot` Entry Command** — interactive kickoff then autonomous handoff
6. **Stop Condition Logic** — triple-check (goals + diminishing returns + max cap)
7. **Project-local config layering** — no existing projects use local `.claude/` dirs; we'd be first

## Constraints

- Claude Code has no persistent background process — loops rely on `/loop` runtime feature or `ScheduleWakeup`
- Agent Mail is the only inter-agent communication channel
- Sub-agents spawned via `Task()` run to completion — no pause/resume
- Token costs scale with number of agents and context size per loop iteration
