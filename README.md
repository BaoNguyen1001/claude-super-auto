# Claude Super Auto

Autonomous idea-to-implementation workflow loop for Claude Code CLI.

Give it an idea. Walk away. Come back to a built project.

## How It Works

```
/autopilot
  │
  ├─ "What do you want to build?"      ← you answer 3 questions
  ├─ "How will we know it's done?"
  └─ "Any constraints?"
  │
  ▼ fully autonomous from here
  │
  ┌─────────────────────────────────────────┐
  │           ITERATION LOOP                │
  │                                         │
  │  1. DISCUSS                             │
  │     Proposer ←→ Challenger              │
  │     (adversarial debate, 3 rounds max)  │
  │                                         │
  │  2. PLAN                                │
  │     Auto-decompose into tasks           │
  │     (uses /planning skill + beads)      │
  │                                         │
  │  3. BUILD                               │
  │     Workers implement with TDD          │
  │     (uses /orchestrator + workers)      │
  │                                         │
  │  4. EVALUATE                            │
  │     Score on 4 dimensions               │
  │     Check stop conditions               │
  │                                         │
  │  Stop when:                             │
  │  - Goals met (score >= 85)              │
  │  - Diminishing returns (delta <= 5)     │
  │  - Max iterations reached (default: 5)  │
  └─────────────────────────────────────────┘
  │
  ▼
  .autopilot/FINAL.md  ← results
```

## Installation

### As a Claude Code plugin

```bash
claude plugin install claude-super-auto
```

### Manual installation

Copy the agents, skills, and commands into your Claude Code config:

```bash
# Clone
git clone https://github.com/BaoNguyen1001/claude-super-auto.git
cd claude-super-auto

# Copy to global config
cp agents/*.md ~/.claude/agents/
cp -r skills/* ~/.claude/skills/
cp commands/*.md ~/.claude/commands/
```

Or copy the `.claude/` directory into any project for project-local usage.

## Usage

```
/autopilot
```

That's it. The command asks 3 questions, then runs autonomously.

## What's Inside

### Agents

| Agent | Model | Role |
|-------|-------|------|
| `proposer` | sonnet | Expands scope, generates features, argues for additions |
| `challenger` | sonnet | Contracts scope, pokes holes, argues for simplification |
| `evaluator` | opus | Scores iterations on 4 weighted dimensions |

### Skills

| Skill | Purpose |
|-------|---------|
| `discussion` | Adversarial debate protocol with convergence detection |
| `autopilot-loop` | Meta-loop controller: discuss → plan → build → evaluate |

### Commands

| Command | Purpose |
|---------|---------|
| `/autopilot` | Entry point: interactive kickoff → autonomous execution |

## Architecture

### Adversarial Discussion

Instead of a multi-agent panel, the system uses **adversarial pairs**:

- **Proposer** pushes outward (features, ambition)
- **Challenger** pulls inward (feasibility, minimalism)
- They debate using structured messages with `[KEEP]`, `[ADD]`, `[CUT]`, `[MODIFY]` tags
- Convergence is detected automatically (agreement ratio >= 0.8 or no disputes remain)
- Max 3 rounds per discussion

### Evaluation

The evaluator scores each iteration on 4 dimensions:

| Dimension | Weight | What It Measures |
|-----------|--------|-----------------|
| Functionality | 40% | Does it satisfy success criteria? |
| Completeness | 25% | What fraction of criteria are implemented? |
| Quality | 20% | Tests exist and pass, no stubs |
| Runnability | 15% | Can it actually be executed? |

### Triple Stop Condition

The loop stops when ANY of these triggers:

1. **Goals met** — overall score >= 85/100
2. **Diminishing returns** — score delta <= 5 between iterations
3. **Safety cap** — max iterations reached (default: 5)

### State Management

```
.autopilot/
  config.md              ← idea, criteria, thresholds
  STATUS.md              ← current progress (gitignored)
  iteration-N/
    discussion-result.md ← what was debated
    evaluation.json      ← scores and verdict
    summary.md           ← iteration recap
  FINAL.md               ← written when loop terminates
```

## Dependencies

This plugin reuses existing Claude Code global skills:

- `/planning` — for task decomposition
- `/orchestrator` — for multi-agent build coordination
- `worker` agent — for TDD-based implementation
- `br` (beads_rust) — for task tracking
- Agent Mail (MCP) — for inter-agent communication

These must be available in your Claude Code configuration.

## License

MIT
