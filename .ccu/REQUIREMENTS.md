# Requirements — Claude Super Auto

## R1: Autonomous Agent Workflow Loop [active]
Build a local-scope Claude Code configuration that takes an initial idea and autonomously cycles through discuss → plan → build → evaluate until done. No human intervention after kickoff.

## R2: Adversarial Discussion Phase [active]
Sub-agent discussion uses adversarial pairs — a **Proposer** (expands features, ideas, ambition) and a **Challenger** (pokes holes, simplifies, cuts scope). They debate until convergence. No panel of 4+ agents.

## R3: Auto Planning & Decomposition [active]
After discussion converges, auto-invoke the planning pipeline: discovery → synthesis → verification → decomposition into phases → epics → tasks. Reuse existing `/planning` skill and `br` (beads_rust) tooling.

## R4: Auto Building with TDD [active]
Auto-invoke the orchestrator to spawn worker agents that implement tasks using TDD. Reuse existing `/orchestrator` skill and `worker` agent.

## R5: Triple Stop Condition [active]
The loop terminates when ANY of:
- **Goal satisfaction** — success criteria (defined at kickoff) are met
- **Diminishing returns** — evaluator scores delta between iterations below threshold
- **Safety cap** — hard max iteration limit reached (default: 5)

## R6: Hybrid State Management [active]
- **Beads (`br`)** for task tracking (plan/build phases)
- **`.autopilot/` directory** for loop-level state: iteration history, convergence scores, discussion transcripts, evaluation reports

## R7: Interactive Kickoff, Then Autonomous [active]
Entry point `/autopilot` asks 2-3 quick questions (idea, success criteria, constraints), then runs fully autonomously. No config file required.

## R8: Reuse Existing Claude Code Config [active]
Must leverage existing global `~/.claude/` agents, commands, skills — especially:
- `/brainstorming` patterns (for discussion phase structure)
- `/planning` skill (for decomposition)
- `/orchestrator` skill (for multi-agent build coordination)
- `worker` agent (for TDD-based implementation)
- `br` (beads_rust) for task tracking

## R9: Local-Scope Project Config [active]
Ships as a project-local `.claude/` directory (agents, commands, skills, CLAUDE.md) that can be dropped into any repo. Does not modify global config.
