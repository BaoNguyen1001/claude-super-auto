# Approach — Claude Super Auto

## Chosen Approach: Skill-as-Controller (Option A)

A single `autopilot-loop` SKILL.md drives the full loop. Each phase calls sub-skills/agents via `Task()`. State written to `.autopilot/` between phases. Simple, one file owns the loop, easy to debug.

### Why not the alternatives?

- **Recipe-Chain (B)**: `/loop` is interval-based (time), not condition-based. Awkward fit for "loop until done" — requires polling and can't naturally express "evaluate then decide."
- **Multi-Skill Pipeline (C)**: More files, coordination overhead between skills, harder to trace flow. The loop is inherently sequential — extra modularity adds complexity without benefit.

## Architecture

```
/autopilot command (interactive kickoff)
    │
    ▼
autopilot-loop skill (controller)
    │
    ├── iteration 1
    │   ├── discussion skill → proposer + challenger (Agent Mail debate)
    │   ├── /planning skill (existing) → beads
    │   ├── /orchestrator skill (existing) → workers build with TDD
    │   └── evaluator agent → score goals + delta
    │       └── STOP? → goals met / diminishing returns / max cap
    │
    ├── iteration 2 (if not stopped)
    │   ├── discussion skill (with context from iteration 1)
    │   ├── /planning skill (incremental — new beads only)
    │   ├── /orchestrator skill
    │   └── evaluator agent
    │       └── STOP?
    │
    └── ... up to max iterations
```

## File Plan

| File | Purpose | Risk |
|------|---------|------|
| `CLAUDE.md` | Project rules, registers autopilot workflow | LOW |
| `.claude/commands/autopilot.md` | Entry point: asks idea, criteria, constraints → invokes controller | LOW |
| `.claude/agents/proposer.md` | Expands scope, generates features, argues via Agent Mail | LOW |
| `.claude/agents/challenger.md` | Cuts scope, finds flaws, argues via Agent Mail | LOW |
| `.claude/agents/evaluator.md` | Scores goal satisfaction (0-100) + iteration delta | **HIGH** |
| `.claude/skills/discussion/SKILL.md` | Spawns proposer+challenger, mediates 3-5 rounds, convergence detection | **HIGH** |
| `.claude/skills/autopilot-loop/SKILL.md` | Controller: sequences phases, manages iterations, checks stop conditions | MEDIUM |
| `.autopilot/` (runtime) | State: config.md, iteration-N/*, STATUS.md | LOW |

## Risk Map

### HIGH Risk — Spike Required

1. **Discussion Skill (adversarial debate protocol)**
   - No existing pattern for multi-turn adversarial debate between sub-agents
   - Key unknowns: How to detect convergence? How many rounds? How to synthesize output?
   - Spike: Create a minimal proposer + challenger, run 3 rounds via Agent Mail, measure convergence

2. **Evaluator Agent (convergence scoring)**
   - No existing scoring/diminishing-returns pattern
   - Key unknowns: What metrics? How to compare iterations? What threshold = "diminishing returns"?
   - Spike: Define scoring rubric, test with mock iteration data

### MEDIUM Risk

3. **Autopilot Controller (loop skill)**
   - Recipe and session-state patterns exist but multi-phase looping is new
   - Key unknown: How to invoke existing skills (planning, orchestrator) programmatically from another skill
   - Mitigation: Test skill-to-skill invocation with a simple example

4. **Integration with existing planning/orchestrator**
   - Skills exist and work standalone — need to validate they work when called from a wrapper skill
   - Mitigation: Validate during controller spike

### LOW Risk

5. All agent definitions (proposer, challenger) — standard frontmatter pattern
6. `/autopilot` command — standard command pattern
7. `.autopilot/` state management — adapts `.ccu/` pattern
8. Project-local `.claude/` config — supported by Claude Code runtime
9. `CLAUDE.md` — standard project config
