# Spike: Discussion Debate Protocol

## Validated Design

### Message Format
Structured markdown with tagged items: `[KEEP]`, `[ADD]`, `[CUT]`, `[MODIFY]`. Plus `## Confidence`, `## Concessions`, `## Open Questions` sections.

### Round Structure
- Fixed 3 rounds max, early-exit on convergence
- Sequential per round: Proposer reads thread → posts → Challenger reads thread → posts
- Mediator (discussion skill) parses and checks convergence between rounds

### Convergence Detection
```
agreement_ratio = |KEEP items in both| / |total unique items|
open_questions = count of items where one says ADD and other says CUT
```
Stop when: `agreement_ratio >= 0.8` OR `open_questions == 0` OR round 3 completes.

### Synthesis
- `[KEEP]` → included as-is
- `[ADD]` surviving final round → included
- `[MODIFY]` → use last-stated version
- `[CUT]` undefended → dropped
- Disputed (ADD vs CUT in final round) → tagged `[DISPUTED]` for planning to resolve

### Context Passing (iteration 2+)
Seed message includes prior evaluator score, feedback, completed items, remaining items. Directive: "focus on evaluator feedback, don't re-debate settled items."

### Key Learnings
1. Fuzzy item matching needed — normalize lowercase, Jaccard similarity > 0.6, or LLM canonicalization
2. Agent prompt must be explicit about template format — reject and retry once if format violated
3. 3-round cap = 6 Task() spawns per discussion — acceptable cost/quality tradeoff
4. `[DISPUTED]` escape hatch avoids needing tie-breaking logic
