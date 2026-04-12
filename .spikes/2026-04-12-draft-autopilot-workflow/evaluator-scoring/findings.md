# Spike: Evaluator Scoring Design

## Validated Design

### Evaluation Procedure
1. Load `.autopilot/config.md` for success criteria
2. Read `.autopilot/iteration-N/` for current build output
3. Read previous `evaluation.json` for delta comparison
4. Inspect artifacts: code exists, tests exist/pass, functionality works
5. Score each dimension, compute delta, emit verdict

### Scoring Rubric

| Dimension | Weight | What It Measures |
|-----------|--------|-----------------|
| `functionality` | 40% | Do artifacts satisfy stated success criteria? |
| `completeness` | 25% | Fraction of criteria with any implementation |
| `quality` | 20% | Tests exist/pass, no stubs/placeholders |
| `runnability` | 15% | Can artifact be executed? Entry point, deps, no crash |

Each dimension: 0-100. Overall = weighted sum.

### Output JSON Schema
```json
{
  "iteration": N,
  "dimensions": { "<name>": { "score": N, "reasoning": "..." } },
  "overall_score": N,
  "previous_score": N|null,
  "delta": N|null,
  "criteria_checklist": [{ "criterion": "...", "met": true|false }],
  "verdict": "continue"|"stop",
  "stop_reason": null|"goals_met"|"diminishing_returns"|"max_iterations",
  "summary": "..."
}
```

### Threshold Defaults
- `goal_threshold`: 85 (overall_score >= 85 → stop)
- `delta_threshold`: 5 (delta <= 5 → stop, exempt for iteration 1)
- `max_iterations`: 5 (hard cap)

Check order: max iterations → goal threshold → diminishing returns. First match wins.

### Key Learnings
1. Why 85 not 100: LLM evaluator rarely gives 100; 85 = "all criteria met with reasonable quality"
2. Delta of 5 accounts for LLM scoring noise
3. `criteria_checklist` feeds next iteration's discussion with explicit unmet items
4. Controller can override verdict (e.g., always continue if iteration < 2)
5. Evaluator should run tests if possible — runnability scoring is less reliable from code reading alone
