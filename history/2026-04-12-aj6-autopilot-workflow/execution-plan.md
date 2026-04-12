# Execution Plan — Claude Super Auto

## Epic
- **ID**: claude-super-auto-aj6
- **Title**: Claude Super Auto: Autonomous Agent Workflow Loop

## Dependency Graph

```
                    972 (CLAUDE.md)
                   /      |       \
                  /       |        \
               uc4      mer       plu
          (Proposer) (Challenger) (State Schema)
                \       /           |
                 \     /            |
                  k43             zl8
            (Discussion)      (Evaluator)
                   \            /
                    \          /
                     xhu ----/
                 (Controller)
                      |
                     tsv
                  (Command)
                      |
                    aj6
                  (Epic)
```

## Execution Phases

### Phase 0: Foundation (sequential, single worker)
| Bead | Title | File Scope |
|------|-------|------------|
| claude-super-auto-972 | Project CLAUDE.md | `CLAUDE.md` |

**Exit gate**: CLAUDE.md exists and defines project identity, agent/skill/command registrations.

### Phase 1: Agents + State (parallel — 2 tracks)

#### Track A: BlueLake (Discussion Agents)
| Bead | Title | File Scope |
|------|-------|------------|
| claude-super-auto-uc4 | Proposer agent definition | `.claude/agents/proposer.md` |
| claude-super-auto-mer | Challenger agent definition | `.claude/agents/challenger.md` |

**Note**: uc4 and mer have no dependency on each other — BlueLake implements them sequentially within the track but they could also be done in parallel.

#### Track B: GreenCastle (Evaluation Path)
| Bead | Title | File Scope |
|------|-------|------------|
| claude-super-auto-plu | .autopilot/ state schema | Documented in controller skill |
| claude-super-auto-zl8 | Evaluator agent definition | `.claude/agents/evaluator.md` |

**Cross-track dependency**: Both tracks require Phase 0 (CLAUDE.md) complete.

**Exit gate**: All 4 agent/state beads complete. All `.claude/agents/*.md` files exist.

### Phase 2: Skills (parallel — 2 tracks, but converge)

#### Track A: BlueLake (continues)
| Bead | Title | File Scope |
|------|-------|------------|
| claude-super-auto-k43 | Discussion skill | `.claude/skills/discussion/SKILL.md` |

**Blocked by**: uc4 (Proposer) + mer (Challenger) from Phase 1.

#### Track B: GreenCastle (idle)
GreenCastle has no Phase 2 work — zl8 completes in Phase 1.

**Exit gate**: Discussion skill exists with full adversarial debate protocol.

### Phase 3: Controller (sequential)
| Bead | Title | File Scope |
|------|-------|------------|
| claude-super-auto-xhu | Autopilot-loop controller skill | `.claude/skills/autopilot-loop/SKILL.md` |

**Blocked by**: k43 (Discussion) + zl8 (Evaluator) + plu (State Schema).
**This is the convergence point** — both tracks must be complete.

**Exit gate**: Controller skill exists with full iteration loop, stop conditions, and integration points for planning/orchestrator skills.

### Phase 4: Entry Point (sequential)
| Bead | Title | File Scope |
|------|-------|------------|
| claude-super-auto-tsv | /autopilot entry command | `.claude/commands/autopilot.md` |

**Blocked by**: xhu (Controller).

**Exit gate**: `/autopilot` command exists, invokes controller skill.

## Track Summary

| Track | Agent Name | Beads | File Scope |
|-------|-----------|-------|------------|
| A | BlueLake | 972 → uc4 → mer → k43 → xhu → tsv | `CLAUDE.md`, `.claude/agents/proposer.md`, `.claude/agents/challenger.md`, `.claude/skills/discussion/SKILL.md`, `.claude/skills/autopilot-loop/SKILL.md`, `.claude/commands/autopilot.md` |
| B | GreenCastle | plu → zl8 | `.claude/agents/evaluator.md` (state schema documented in controller) |

**Cross-track dependencies**:
- GreenCastle starts after Phase 0 (CLAUDE.md) complete
- Phase 3 (controller) waits for BOTH tracks to complete

## File Scope (no overlap)

| Track | Files |
|-------|-------|
| BlueLake | `CLAUDE.md`, `.claude/agents/proposer.md`, `.claude/agents/challenger.md`, `.claude/skills/discussion/SKILL.md`, `.claude/skills/autopilot-loop/SKILL.md`, `.claude/commands/autopilot.md` |
| GreenCastle | `.claude/agents/evaluator.md` |

## Key Learnings from Spikes

### Discussion Protocol (embed in k43 bead)
- Message format: tagged items [KEEP/ADD/CUT/MODIFY] + Confidence + Concessions + Open Questions
- 3 rounds max, early-exit on convergence (agreement_ratio >= 0.8 OR open_questions == 0)
- Synthesis: KEEP→in, surviving ADD→in, MODIFY→last version, undefended CUT→drop, disputed→[DISPUTED]
- Context passing for iteration 2+: include evaluator score, feedback, completed/remaining items

### Evaluator Scoring (embed in zl8 bead)
- 4 dimensions: functionality (40%), completeness (25%), quality (20%), runnability (15%)
- Output: structured JSON with dimensions, overall_score, delta, criteria_checklist, verdict
- Thresholds: goal=85, delta=5, max_iterations=5
- Check order: max iterations → goal → diminishing returns

## Estimated Execution

- Phase 0: 1 bead (sequential)
- Phase 1: 4 beads (2 parallel tracks)
- Phase 2: 1 bead (sequential on Track A)
- Phase 3: 1 bead (sequential, convergence)
- Phase 4: 1 bead (sequential)

Total: 8 beads, 2 parallel tracks, ~4 effective sequential phases.
