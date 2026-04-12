# Claude Super Auto

Autonomous agent workflow loop for Claude Code CLI. Takes an initial idea and autonomously cycles through discuss → plan → build → evaluate until done.

## How It Works

1. User runs `/autopilot` and provides an idea, success criteria, and optional constraints
2. The system enters a fully autonomous loop:
   - **Discussion**: Adversarial Proposer + Challenger agents debate scope and features
   - **Planning**: Invokes the global `/planning` skill to decompose into beads
   - **Building**: Invokes the global `/orchestrator` skill to spawn workers with TDD
   - **Evaluation**: Evaluator agent scores progress against success criteria
3. Loop continues until: goals met (score >= 85) OR diminishing returns (delta <= 5) OR max iterations (default 5)

## Project-Local Agents

| Agent | File | Purpose |
|-------|------|---------|
| `proposer` | `.claude/agents/proposer.md` | Expands scope, generates features during discussion phase |
| `challenger` | `.claude/agents/challenger.md` | Cuts scope, challenges feasibility during discussion phase |
| `evaluator` | `.claude/agents/evaluator.md` | Scores iterations on 4 dimensions, checks convergence |

## Project-Local Skills

| Skill | File | Purpose |
|-------|------|---------|
| `discussion` | `.claude/skills/discussion/SKILL.md` | Adversarial debate protocol with convergence detection |
| `autopilot-loop` | `.claude/skills/autopilot-loop/SKILL.md` | Meta-loop controller sequencing all phases |

## Project-Local Commands

| Command | File | Purpose |
|---------|------|---------|
| `/autopilot` | `.claude/commands/autopilot.md` | Entry point: interactive kickoff → autonomous execution |

## Reused Global Skills

These existing global skills are invoked during the loop — do NOT duplicate them:

- `/planning` — discovery → synthesis → verification → decomposition → track planning
- `/orchestrator` — spawns parallel worker agents, Agent Mail coordination
- `worker` agent — TDD-based bead implementation
- `oracle` agent — advisory reasoning for synthesis and evaluation
- `code-reviewer` agent — post-build quality sweep

## Runtime State: `.autopilot/`

The `.autopilot/` directory tracks loop state across iterations:

```
.autopilot/
  config.md              — idea, success criteria, constraints, thresholds
  STATUS.md              — current iteration, latest scores (gitignored)
  iteration-N/
    discussion-result.md — synthesized discussion output
    evaluation.json      — evaluator scores and verdict
    summary.md           — iteration summary
  FINAL.md               — written when loop terminates
```

## Conventions

- Agent/skill definitions use markdown with YAML frontmatter (`name`, `description`, `model`)
- Commands use plain markdown with numbered steps (no frontmatter)
- All inter-agent communication uses Agent Mail (MCP) with thread-based messaging
- Beads (`br`) for task tracking, Beads Viewer (`bv`) for dependency analysis
