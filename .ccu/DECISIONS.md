# Decisions — Claude Super Auto

## D1: Fully Autonomous Execution (2026-04-12)
**Decision:** No human-in-the-loop after kickoff. Agents make all decisions autonomously.
**Why:** The core value proposition is "idea in, project out." Any pause for human approval breaks the autonomous loop and defeats the purpose. The triple stop condition (D3) provides the safety net instead of human gates.

## D2: Adversarial Pairs for Discussion (2026-04-12)
**Decision:** Discussion phase uses exactly 2 sub-agents — Proposer and Challenger — not a multi-agent panel.
**Why:** Fewer agents produce higher-signal output. A 4+ agent panel dilutes opinions, creates coordination overhead (who breaks ties?), and explodes token cost. Adversarial pairs converge naturally — when neither can move the other, the discussion is done. In later loops, this maps directly to "what to add next?" vs. "is this good enough?" which feeds the diminishing-returns stop condition.

## D3: Triple Stop Condition (2026-04-12)
**Decision:** Loop stops on goal satisfaction OR diminishing returns OR safety cap (whichever triggers first).
**Why:** Goal-based alone can't detect when improvements are marginal. Diminishing-returns alone can't detect when goals are actually met. Safety cap prevents runaway execution if the other two conditions never trigger. All three together cover each other's blind spots.

## D4: Hybrid State — Beads + .autopilot/ (2026-04-12)
**Decision:** Use beads for task tracking (plan/build), `.autopilot/` directory for loop-level state (iteration history, scores, transcripts).
**Why:** Beads are designed for task decomposition and tracking — great for plan/build. But they have no concept of "loop iteration" or "convergence score." A separate `.autopilot/` directory holds the meta-state that spans across beads iterations. Both are file-based and git-trackable.

## D5: Interactive Kickoff Command (2026-04-12)
**Decision:** `/autopilot` command asks 2-3 quick questions then runs autonomously. No config file.
**Why:** Config files add friction and setup overhead. Inline-only misses structured input (success criteria). Interactive kickoff gets structured input naturally through conversation, then hands off to full autonomy. Best UX for the "idea in, project out" flow.

## D6: Reuse Global Config, Ship as Local (2026-04-12)
**Decision:** The framework is a project-local `.claude/` config that references and extends existing global skills/agents.
**Why:** Global config already has high-quality orchestrator, planning, worker, brainstorming assets. Duplicating them would create maintenance burden and drift. Local config adds only the autopilot-specific pieces (loop controller, evaluator, discussion agents, `/autopilot` command).
