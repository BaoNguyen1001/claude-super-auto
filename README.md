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
  │  4.5 REVIEW                             │
  │     Regression detection                │
  │     (code-reviewer + evaluator)         │
  │                                         │
  │  Stop when (bounded mode):              │
  │  - Goals met (score >= 85)              │
  │  - Diminishing returns (delta <= 5)     │
  │  - Max iterations reached (default: 5)  │
  │                                         │
  │  Stop when (unlimited mode):            │
  │  - Goals met (score >= 95)              │
  │  - Session interrupted by user          │
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

### Two Modes

| Mode | Goal Threshold | Delta | Max Iterations | Behavior |
|------|---------------|-------|----------------|----------|
| **Bounded** (default) | 85% | 5 | 5 | Standard loop with triple stop condition |
| **Unlimited** | 95% | 3 (focus shift) | None | Runs until goals met or session interrupted. AI-driven action selection each iteration. |

In **unlimited mode**, the agent autonomously decides each iteration's focus (build, test, review, refactor, fix bugs, upgrade architecture, etc.) based on full project state analysis. Diminishing returns triggers a strategy shift instead of stopping.

### Adversarial Discussion

Instead of a multi-agent panel, the system uses **strongly differentiated adversarial pairs**:

- **Proposer** (ambitious product visionary) — optimizes for user value and impact
- **Challenger** (battle-scarred staff engineer) — optimizes for engineering cost and risk
- They debate using structured messages with `[KEEP]`, `[ADD]`, `[CUT]`, `[MODIFY]` tags
- No concessions in Round 1 — agents stake positions first, then negotiate
- Convergence is detected automatically (agreement ratio >= 0.8 or no disputes remain)
- Max 3 rounds per discussion

### Regression-Aware Review

After building and before evaluation, a **regression review** phase:

1. **Code-reviewer agent** traces changed files, identifies dependents, checks for breakage
2. **Evaluator** compares criteria checklist against previous iteration — any regression from `met→unmet` is penalized 15 points
3. Regressions found during review are fixed before evaluation scores them

### Evaluation

The evaluator scores each iteration on 4 dimensions:

| Dimension | Weight | What It Measures |
|-----------|--------|-----------------|
| Functionality | 40% | Does it satisfy success criteria? |
| Completeness | 25% | What fraction of criteria are implemented? |
| Quality | 20% | Tests exist and pass, no stubs |
| Runnability | 15% | Can it actually be executed? |

The evaluator can invoke global skills (`qa-sweep`, `security-review`, `ubs`) to ground scores in tool output.

### Stop Conditions

**Bounded mode** — stops when ANY triggers:
1. **Goals met** — overall score >= 85/100
2. **Diminishing returns** — score delta <= 5 between iterations
3. **Safety cap** — max iterations reached (default: 5)

**Unlimited mode** — only stops when:
1. **Goals met** — overall score >= 95/100
2. **Session interrupted** — user breaks the session

Diminishing returns in unlimited mode triggers a **focus shift** — the agent changes strategy instead of stopping.

### State Management

```
.autopilot/
  config.md              ← idea, criteria, thresholds, mode
  STATUS.md              ← current progress (gitignored)
  HISTORY.md             ← consolidated cross-iteration log (committed, drives resume)
  iteration-N/
    action-plan.md       ← AI-driven action selection (unlimited mode)
    discussion-result.md ← what was debated
    review-result.md     ← regression review findings
    evaluation.json      ← scores, verdict, regressions
    summary.md           ← iteration recap
  archive/{timestamp}/   ← snapshot of prior session (written on "Start fresh")
  FINAL.md               ← written when loop terminates
```

### Resume After a Break

If you stop a session mid-loop and later run `/autopilot` again, it detects the prior state in `.autopilot/STATUS.md` and asks:

- **Resume** — continues from the next incomplete iteration, reusing the idea, criteria, and mode stored in `config.md`. `HISTORY.md` is passed to the Proposer/Challenger so they do not re-argue scope you already settled.
- **Start fresh** — archives the prior session to `.autopilot/archive/{timestamp}/` and kicks off a new loop.

Mid-iteration crashes are recovered the same way — phase artifacts already on disk are skipped so no work is repeated.

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
