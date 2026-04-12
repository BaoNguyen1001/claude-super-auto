---
name: autopilot-loop
description: Central controller for the autonomous workflow loop. Sequences discuss → plan → build → evaluate phases and manages iteration state. Invoked by the /autopilot command after interactive kickoff.
model: opus
---

# Autopilot Loop Controller

Autonomous meta-loop that takes a configured idea and iteratively refines it through discussion, planning, building, and evaluation until stop conditions are met.

## Prerequisites

Before this skill runs, the `/autopilot` command must have:
1. Written `.autopilot/config.md` with idea, success criteria, constraints, and thresholds
2. Initialized `.autopilot/STATUS.md`

## Configuration

Read thresholds from `.autopilot/config.md`:

```
GOAL_THRESHOLD = config.goal_threshold       # default: 85
DELTA_THRESHOLD = config.delta_threshold     # default: 5
MAX_ITERATIONS = config.max_iterations       # default: 5
IDEA = config.idea
SUCCESS_CRITERIA = config.success_criteria
CONSTRAINTS = config.constraints
```

---

## Main Loop

```
for ITERATION in 1..MAX_ITERATIONS:

    update_status("iteration": ITERATION, "phase": "starting")

    # ── Phase 1: Discussion ──────────────────────────────
    run_discussion(ITERATION)

    # ── Phase 2: Planning ────────────────────────────────
    run_planning(ITERATION)

    # ── Phase 3: Building ────────────────────────────────
    run_building(ITERATION)

    # ── Phase 4: Evaluation ──────────────────────────────
    evaluation = run_evaluation(ITERATION)

    # ── Phase 5: Stop Check ──────────────────────────────
    should_stop, reason = check_stop_conditions(evaluation, ITERATION)

    # ── Phase 6: Write Summary ───────────────────────────
    write_iteration_summary(ITERATION, evaluation, should_stop, reason)

    if should_stop:
        write_final_report(reason)
        break

# If loop exhausted without stopping
if ITERATION == MAX_ITERATIONS and not should_stop:
    write_final_report("max_iterations")
```

---

## Phase 1: Discussion

Invoke the discussion skill to run an adversarial Proposer/Challenger debate.

```
Skill(skill="discussion")
```

**Inputs to provide** (set as context before invoking):

| Input | Iteration 1 | Iteration 2+ |
|-------|------------|--------------|
| ITERATION | Current iteration number | Current iteration number |
| IDEA | From config | From config |
| SUCCESS_CRITERIA | From config | From config |
| CONSTRAINTS | From config | From config |
| PRIOR_EVALUATION | N/A | Previous evaluation.json |
| COMPLETED_ITEMS | N/A | Items with `met: true` in previous criteria_checklist |
| REMAINING_ITEMS | N/A | Items with `met: false` in previous criteria_checklist |

**Output**: `.autopilot/iteration-{ITERATION}/discussion-result.md`

**Verify**: File exists before proceeding. If discussion skill fails:
1. Log error to STATUS.md
2. Use previous iteration's discussion result (if exists) or the raw idea as fallback
3. Proceed to planning — do not halt the loop for a discussion failure

---

## Phase 2: Planning

Invoke the global planning skill to decompose the discussion result into executable beads.

```
Skill(skill="planning")
```

**Context to provide**:
- Read `.autopilot/iteration-{ITERATION}/discussion-result.md` as the "user request"
- Include the success criteria from config
- For iteration 2+: include what was already built (from prior iteration summary)

**Output**: Beads in `.beads/`, execution plan in `history/` directory

**Special handling for iteration 2+**:
- Planning should create ONLY new beads for unmet criteria and discussion additions
- It should NOT re-plan already-completed work
- Reference prior iteration's beads to avoid duplication: `br list --closed`

**Verify**: At least one bead exists in open/ready state. If planning produces zero new beads:
- This means all criteria are likely met — skip building, proceed to evaluation
- Log "No new work planned" in summary

---

## Phase 3: Building

Invoke the global orchestrator skill to spawn workers and execute beads.

```
Skill(skill="orchestrator")
```

**Context**: The orchestrator reads the execution plan produced by Phase 2 and spawns worker agents.

**Output**: Committed code/config, closed beads

**Verify**: Check that beads were closed:
```bash
br stats --json | jq '.summary.closed_issues'
```

**If building fails** (workers stuck, beads not closing):
1. Log failure details to `.autopilot/iteration-{ITERATION}/build-errors.md`
2. Proceed to evaluation anyway — the evaluator will score the incomplete work
3. The low score will either trigger another iteration or stop via diminishing returns

---

## Phase 4: Evaluation

Spawn the evaluator agent to score this iteration's results.

```
mkdir -p .autopilot/iteration-{ITERATION}

evaluator_result = Task(
    subagent_type="evaluator",
    model="opus",
    prompt="Evaluate iteration {ITERATION}. Read .autopilot/config.md for success criteria. Write your evaluation JSON to .autopilot/iteration-{ITERATION}/evaluation.json"
)
```

**Output**: `.autopilot/iteration-{ITERATION}/evaluation.json`

**Verify**: File exists and contains valid JSON with required fields:
- `overall_score` (number 0-100)
- `delta` (number or null)
- `verdict` ("continue" or "stop")
- `criteria_checklist` (array)

**If evaluation fails**: Assign `overall_score = 0, verdict = "continue"` and log the failure.

---

## Phase 5: Stop Condition Check

Check conditions in this specific order. **First match wins.**

```
function check_stop_conditions(evaluation, iteration):

    # 1. Safety cap (always checked first)
    if iteration >= MAX_ITERATIONS:
        return (true, "max_iterations")

    # 2. Goal satisfaction
    if evaluation.overall_score >= GOAL_THRESHOLD:
        return (true, "goals_met")

    # 3. Diminishing returns (skip for iterations 1-2 — too early to judge)
    if iteration >= 3 and evaluation.delta is not null:
        if evaluation.delta <= DELTA_THRESHOLD:
            return (true, "diminishing_returns")

    # 4. Controller override: always continue if iteration < 2
    #    (early scores are unreliable — scaffolding iteration)
    if iteration < 2:
        return (false, null)

    # 5. Default: respect evaluator's verdict
    return (evaluation.verdict == "stop", evaluation.stop_reason)
```

### Override Logic

The controller can override the evaluator in specific cases:

| Situation | Override | Why |
|-----------|----------|-----|
| Iteration 1, score < 85 | Always continue | First iteration is scaffolding — score is artificially low |
| Iteration 1, score >= 85 | Respect (stop) | If somehow everything is done in one shot, accept it |
| Delta <= 5 but iteration < 3 | Continue | Give at least 3 iterations before stopping on diminishing returns |
| Delta <= 5 but iteration < 3 | Continue | Give at least 3 iterations before stopping on diminishing returns |

---

## Phase 6: Write Iteration Summary

After evaluation, write a summary of this iteration.

**Write to** `.autopilot/iteration-{ITERATION}/summary.md`:

```markdown
# Iteration {ITERATION} Summary

## Score: {overall_score}/100 (delta: {delta})
## Verdict: {verdict} ({stop_reason or "continuing"})

## What Was Discussed
- Agreed features: {count from discussion-result}
- Disputed features: {count}
- Cut features: {count}

## What Was Built
- Beads completed: {count}
- Commits: {list of hashes}
- Key files: {list}

## Criteria Status
{for each criterion}
- {'[x]' if met else '[ ]'} {criterion}

## What's Next
{if continuing: "Proceeding to iteration {ITERATION+1}. Focus areas: {unmet criteria}"}
{if stopping: "Loop terminated: {stop_reason}"}
```

**Update** `.autopilot/STATUS.md`:

```markdown
# Autopilot Status

- Current iteration: {ITERATION}
- Latest score: {overall_score}/100
- Status: {running|completed}
- Stop reason: {reason or "in progress"}
- Last updated: {timestamp}
```

---

## Loop Termination: Final Report

When the loop stops (any reason), write `.autopilot/FINAL.md`:

```markdown
# Autopilot Final Report

## Configuration
- Idea: {IDEA}
- Success criteria: {count} defined
- Max iterations: {MAX_ITERATIONS}
- Thresholds: goal={GOAL_THRESHOLD}, delta={DELTA_THRESHOLD}

## Result
- Iterations completed: {ITERATION}
- Stop reason: {reason}
- Final score: {overall_score}/100

## Criteria Completion
{for each criterion}
- {'[x]' if met else '[ ]'} {criterion}

## Score Progression
| Iteration | Score | Delta | Verdict |
|-----------|-------|-------|---------|
{for each iteration}
| {N} | {score} | {delta} | {verdict} |

## Iteration Summaries
{for each iteration, brief 2-3 line summary}

## Files Created
{list all files created/modified across all iterations}
```

Then:
1. Update STATUS.md with `Status: completed`
2. Commit all `.autopilot/` artifacts:
```bash
git add .autopilot/
git commit -m "autopilot: complete — {stop_reason} after {ITERATION} iterations (score: {overall_score}/100)"
```

---

## Error Recovery

| Failure | Recovery |
|---------|----------|
| Discussion skill fails | Use raw idea or prior discussion result as planning input |
| Planning produces no beads | Skip build, go to evaluation (likely goals already met) |
| Building fails mid-execution | Proceed to evaluation — low score triggers next iteration |
| Evaluation fails | Score 0, verdict "continue" — ensures another attempt |
| Agent Mail unavailable | Skills fall back to direct Task() prompts without thread persistence |
| `.autopilot/` directory missing | Re-create from config and continue |

### Crash Recovery

If the loop crashes mid-iteration:
1. Read `.autopilot/STATUS.md` for last known state
2. Check which phase artifacts exist for the current iteration
3. Resume from the next incomplete phase (don't re-run completed phases)

---

## State File Reference

| File | Written By | Read By | Purpose |
|------|-----------|---------|---------|
| `.autopilot/config.md` | /autopilot command | All phases | Idea, criteria, thresholds |
| `.autopilot/STATUS.md` | Controller (this) | Crash recovery | Current state |
| `.autopilot/iteration-N/discussion-result.md` | Discussion skill | Planning phase | What to build |
| `.autopilot/iteration-N/evaluation.json` | Evaluator agent | Controller, next discussion | Scores and verdict |
| `.autopilot/iteration-N/summary.md` | Controller (this) | Human review | Iteration recap |
| `.autopilot/FINAL.md` | Controller (this) | Human review | Final report |
