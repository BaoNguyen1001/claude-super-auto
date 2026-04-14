---
name: autopilot-loop
description: Central controller for the autonomous workflow loop. Generates ideas via /t:top-ideas, auto-discusses via /t:discuss, plans, builds, and evaluates — all without user input.
model: opus
---

# Autopilot Loop Controller

Fully autonomous meta-loop. The user provided only an initial idea — this controller handles everything else: idea generation, discussion, planning, building, and evaluation.

## Prerequisites

Before this skill runs, the `/autopilot` command must have:
1. Written `.autopilot/config.md` with idea, auto-generated success criteria, and thresholds
2. Initialized `.autopilot/STATUS.md`

## Configuration

Read from `.autopilot/config.md`:

```
MODE = config.mode                           # "bounded" or "unlimited"
GOAL_THRESHOLD = config.goal_threshold       # bounded: 85, unlimited: 95
DELTA_THRESHOLD = config.delta_threshold     # bounded: 5, unlimited: 3
MAX_ITERATIONS = config.max_iterations       # bounded: 5, unlimited: null
IDEA = config.idea
SUCCESS_CRITERIA = config.success_criteria   # auto-generated
CONSTRAINTS = config.constraints             # auto-inferred
```

---

## Main Loop

```
ITERATION = 0

while true:
    ITERATION += 1

    # Bounded mode: enforce max iterations
    if MODE == "bounded" and ITERATION > MAX_ITERATIONS:
        write_final_report("max_iterations")
        break

    update_status("iteration": ITERATION, "phase": "starting")

    # ── Phase 0: Action Selection (unlimited mode) ─────
    if MODE == "unlimited":
        action_plan = run_ai_action_selection(ITERATION)
    
    # ── Phase 1: Idea Generation ────────────────────────
    ideas = generate_top_ideas(ITERATION)

    # ── Phase 2: Auto-Discussion ────────────────────────
    run_auto_discussion(ITERATION, ideas)

    # ── Phase 3: Planning ───────────────────────────────
    run_planning(ITERATION)

    # ── Phase 4: Building ───────────────────────────────
    run_building(ITERATION)

    # ── Phase 4.5: Regression Review ────────────────────
    run_regression_review(ITERATION)

    # ── Phase 5: Evaluation ─────────────────────────────
    evaluation = run_evaluation(ITERATION)

    # ── Phase 6: Stop Check ─────────────────────────────
    should_stop, reason = check_stop_conditions(evaluation, ITERATION)

    # ── Phase 7: Write Summary ──────────────────────────
    write_iteration_summary(ITERATION, evaluation, should_stop, reason)

    if should_stop:
        write_final_report(reason)
        break

# Bounded mode: if loop exhausted without stopping
if MODE == "bounded" and ITERATION >= MAX_ITERATIONS and not should_stop:
    write_final_report("max_iterations")

# Unlimited mode: loop only exits via break (goals_met or user interrupt)
# If the session is interrupted, crash recovery will write FINAL.md on next resume
```

---

## Phase 0: AI-Driven Action Selection (Unlimited Mode Only)

**Skip this phase in bounded mode.** In bounded mode, proceed directly to Phase 1.

In unlimited mode, the agent autonomously decides what to focus on each iteration by analyzing the full project state. This replaces rigid phase ordering with intelligent, context-aware task selection.

### State Analysis

Before deciding, gather signals from ALL of these sources:

```bash
# 1. Evaluator scores from previous iteration (if exists)
cat .autopilot/iteration-$((ITERATION-1))/evaluation.json 2>/dev/null | jq '{overall_score, delta, dimensions, regression_count}'

# 2. Test results
npm test 2>&1 | tail -20 || python -m pytest --tb=short 2>&1 | tail -20

# 3. Code quality signals
grep -r "TODO\|FIXME\|placeholder\|stub\|NotImplemented" --include="*.ts" --include="*.js" --include="*.py" -c . 2>/dev/null | head -10

# 4. Unmet success criteria
cat .autopilot/config.md  # review criteria
cat .autopilot/iteration-$((ITERATION-1))/evaluation.json 2>/dev/null | jq '.criteria_checklist[] | select(.met == false)'

# 5. Recent changes and potential impacts
git log --oneline -10
git diff HEAD~3..HEAD --stat 2>/dev/null

# 6. Regression review from last iteration
cat .autopilot/iteration-$((ITERATION-1))/review-result.md 2>/dev/null
```

### Action Categories

Based on the analysis, select ONE OR MORE actions for this iteration:

| Action | When to Choose | Signals |
|--------|---------------|---------|
| **build-features** | Unmet success criteria need new code | Low completeness score, criteria with `met: false` |
| **fix-bugs** | Tests failing, regressions detected | Failed tests, `regression_count > 0`, error logs |
| **refactor** | Code quality is low, high complexity | Low quality score, many TODOs/stubs, code smells |
| **review-code** | Recent large changes, low quality score | Many files changed, quality dimension < 60 |
| **test** | Low test coverage, untested features | Quality score low, tests absent for key features |
| **upgrade-architecture** | Structural issues blocking progress | Diminishing returns (delta <= 3) with low scores, repeated failures in same area |
| **apply-patterns** | Code inconsistency, framework misuse | Quality issues, convention violations |
| **security-review** | Code handles user input, auth, APIs | New endpoints, auth changes, input handling |
| **polish** | High scores but unmet criteria remain | Overall score > 80 but some criteria still unmet |

### Decision Rules

1. **If regressions exist**: ALWAYS include `fix-bugs` — regressions take priority
2. **If tests are failing**: Include `fix-bugs` before adding new features
3. **If delta <= 3 (diminishing returns)**: Switch strategy — if you were building, try refactoring or upgrading architecture instead. Do NOT repeat the same action type that produced low delta.
4. **If overall score > 80**: Focus on `polish`, `test`, or `review-code` rather than new features
5. **Multi-action**: You MAY combine 2-3 compatible actions (e.g., `build-features` + `test`, or `refactor` + `review-code`)

### Output

Store the action plan in `.autopilot/iteration-{ITERATION}/action-plan.md`:

```markdown
# Action Plan — Iteration {ITERATION}

## State Analysis
- Previous score: {score}/100 (delta: {delta})
- Test status: {passing/failing/absent}
- Regressions: {count}
- Unmet criteria: {count}
- Code quality signals: {summary}

## Selected Actions
1. {PRIMARY_ACTION} — {rationale}
2. {SECONDARY_ACTION} — {rationale} (if applicable)

## Focus Areas
- {specific files, modules, or criteria to target}

## Strategy Shift
{if delta was low: "Previous approach ({last_action}) produced low delta. Shifting to {new_action} to break plateau."}
{if no shift needed: "Continuing current trajectory."}
```

The action plan informs Phase 1 (idea generation) and Phase 2 (discussion) — the Proposer and Challenger debate WITHIN the scope of the selected actions, not open-endedly.

---

## Phase 1: Idea Generation

Use `/t:top-ideas` to autonomously generate improvement ideas based on the current state of the project and the original idea.

In **unlimited mode**, scope idea generation to the selected actions from Phase 0. Pass the action plan as context so ideas align with the chosen focus.

```
Skill(skill="t:top-ideas")
```

**Context to provide** (set before invoking):

| Input | Iteration 1 | Iteration 2+ |
|-------|------------|--------------|
| IDEA | From config | From config |
| Focus area | Original idea scope | Unmet criteria + evaluator feedback |
| Current state | Codebase as-is | What was built in prior iterations |
| PRIOR_ITERATION_HISTORY | N/A | Full contents of `.autopilot/HISTORY.md` — ideas marked done here MUST NOT be re-proposed |

**Post-processing**: Parse the output to extract a ranked list of ideas. Store to `.autopilot/iteration-{ITERATION}/top-ideas.md`.

**Auto-pick**: Select the top 3-5 ideas that are:
1. Most aligned with the original `IDEA` and unmet `SUCCESS_CRITERIA`
2. Feasible within a single iteration
3. Not duplicating already-completed work — cross-check against `.autopilot/HISTORY.md` (authoritative, consolidated) and individual `iteration-N/summary.md` files

Store selected ideas in `.autopilot/iteration-{ITERATION}/selected-ideas.md`.

**If `/t:top-ideas` fails**: Fall back to deriving ideas directly from:
- Unmet success criteria (iteration 2+)
- The original idea broken into logical components (iteration 1)

---

## Phase 2: Auto-Discussion

Use `/t:discuss` to autonomously run a guided requirements gathering and adversarial debate on the selected ideas. This replaces manual user input — the agent provides all context.

```
Skill(skill="t:discuss")
```

**Context to provide**: Frame the discussion around the selected ideas from Phase 1. The agent acts as the "user" providing requirements — no human input needed.

| Input | Value |
|-------|-------|
| Topic | "Implement the following ideas for: {IDEA}" |
| Selected ideas | From `.autopilot/iteration-{ITERATION}/selected-ideas.md` |
| Success criteria | From config (unmet ones for iteration 2+) |
| Constraints | From config |
| Prior work | What was already built (iteration 2+) |

If `/t:discuss` is not available, fall back to the project-local discussion skill:

```
Skill(skill="discussion")
```

When falling back to the local discussion skill, provide these inputs:

| Input | Iteration 1 | Iteration 2+ |
|-------|------------|--------------|
| ITERATION | Current iteration number | Current iteration number |
| IDEA | From config | From config |
| SUCCESS_CRITERIA | From config | From config |
| CONSTRAINTS | From config | From config |
| SELECTED_IDEAS | From Phase 1 | From Phase 1 |
| PRIOR_EVALUATION | N/A | Previous evaluation.json |
| COMPLETED_ITEMS | N/A | Items with `met: true` in previous criteria_checklist |
| REMAINING_ITEMS | N/A | Items with `met: false` in previous criteria_checklist |
| PRIOR_ITERATION_HISTORY | N/A | Full contents of `.autopilot/HISTORY.md` — Proposer and Challenger MUST NOT re-argue scope already resolved in prior iterations |

**Output**: `.autopilot/iteration-{ITERATION}/discussion-result.md`

**Verify**: File exists before proceeding. If discussion fails:
1. Log error to STATUS.md
2. Use the selected ideas from Phase 1 directly as planning input (skip debate)
3. Proceed to planning — do not halt the loop for a discussion failure

---

## Phase 3: Planning

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

## Phase 4: Building

Invoke the global orchestrator skill to spawn workers and execute beads.

```
Skill(skill="orchestrator")
```

**Context**: The orchestrator reads the execution plan produced by Phase 3 and spawns worker agents.

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

## Phase 4.5: Regression Review

After building and before evaluation, spawn the global `code-reviewer` agent to detect regressions — new code that may have broken existing features.

### Record Pre-Build Baseline

Before Phase 4 begins, capture the commit hash so the review can diff against it:

```bash
PRE_BUILD_COMMIT=$(git rev-parse HEAD)
```

### Spawn Regression Reviewer

```
review_result = Task(
    subagent_type="code-reviewer",
    prompt="""
    Regression review for autopilot iteration {ITERATION}.

    ## Your Task
    Detect whether the code changes in this iteration broke any existing functionality.

    ## Steps

    1. **Identify changed files**:
       ```bash
       git diff {PRE_BUILD_COMMIT}..HEAD --name-only
       ```

    2. **Trace dependents**: For each changed file, find files that import or reference it:
       ```bash
       grep -rl "import.*{module_name}" --include="*.ts" --include="*.js" --include="*.py" . 2>/dev/null
       grep -rl "{function_name}" --include="*.ts" --include="*.js" --include="*.py" . 2>/dev/null
       ```

    3. **Check for breakage**:
       - Removed or renamed exports that other files depend on
       - Changed function signatures (added required params, changed return types)
       - Broken imports (files importing something that no longer exists)
       - Modified shared state or config that other modules rely on

    4. **Run full test suite** and compare results:
       ```bash
       npm test 2>&1 || python -m pytest --tb=short 2>&1 || echo "No test runner found"
       ```

    5. **Report**: Write findings to `.autopilot/iteration-{ITERATION}/review-result.md`:
       ```markdown
       # Regression Review — Iteration {ITERATION}

       ## Files Changed
       - <list>

       ## Dependents Checked
       - <file>: <dependents found>

       ## Regressions Found
       - <description of each regression, or "None detected">

       ## Test Results
       - Total: N, Passed: N, Failed: N

       ## Recommended Fixes
       - <fix description, or "No fixes needed">
       ```

    6. **If regressions found**: Attempt to fix them directly. For each regression:
       - Read the broken file and the change that caused the break
       - Apply the minimal fix (update imports, adjust function calls, etc.)
       - Re-run affected tests to verify the fix
       - Commit fixes with message: "fix: regression from iteration {ITERATION} — <description>"
    """
)
```

**Output**: `.autopilot/iteration-{ITERATION}/review-result.md`

**Verify**: File exists. If review finds regressions and fixes them, verify the fix commits exist.

**If review fails** (agent error, timeout):
1. Log failure to `.autopilot/iteration-{ITERATION}/review-result.md`: "Review skipped: {error}"
2. Proceed to evaluation — the evaluator has its own regression detection as a fallback

---

## Phase 5: Evaluation

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

## Phase 6: Stop Condition Check

Check conditions in this specific order. **First match wins.** Behavior differs between bounded and unlimited modes.

### Bounded Mode

```
function check_stop_conditions_bounded(evaluation, iteration):

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
    if iteration < 2:
        return (false, null)

    # 5. Default: respect evaluator's verdict
    return (evaluation.verdict == "stop", evaluation.stop_reason)
```

### Unlimited Mode

In unlimited mode, the loop NEVER stops on `max_iterations` or `diminishing_returns`. Diminishing returns triggers a **focus shift** instead.

```
function check_stop_conditions_unlimited(evaluation, iteration):

    # 1. Goal satisfaction — the ONLY auto-stop condition
    if evaluation.overall_score >= GOAL_THRESHOLD:  # 95%
        return (true, "goals_met")

    # 2. Diminishing returns — SHIFT FOCUS, do not stop
    if iteration >= 3 and evaluation.delta is not null:
        if evaluation.delta <= DELTA_THRESHOLD:  # 3
            # Signal to Phase 0 in the NEXT iteration to change strategy
            write_focus_shift_signal(iteration, evaluation)
            # Continue — do NOT stop
            return (false, null)

    # 3. Controller override: always continue if iteration < 2
    if iteration < 2:
        return (false, null)

    # 4. Default: always continue (unlimited mode never stops on evaluator verdict)
    return (false, null)
```

### Focus Shift Signal (Unlimited Mode)

When diminishing returns are detected, write `.autopilot/iteration-{ITERATION}/focus-shift.md`:

```markdown
# Focus Shift Required

Previous delta: {delta} (below threshold {DELTA_THRESHOLD})
Previous action types: {actions from last 2 iterations}

The current approach is producing diminishing returns.
Phase 0 in the next iteration MUST select a DIFFERENT action category.

Suggested shifts:
- If last action was build-features → try refactor or upgrade-architecture
- If last action was refactor → try test or review-code
- If last action was test → try build-features or polish
- If last action was review-code → try build-features or fix-bugs
```

### Override Logic

| Mode | Situation | Override | Why |
|------|-----------|----------|-----|
| Both | Iteration 1, score < threshold | Always continue | First iteration is scaffolding |
| Both | Iteration 1, score >= threshold | Respect (stop) | Done in one shot |
| Bounded | Delta <= 5 but iteration < 3 | Continue | Too early to judge |
| Unlimited | Delta <= 3 | **Shift focus, continue** | Change strategy instead of stopping |
| Unlimited | Max iterations | **Ignored** | No cap in unlimited mode |

---

## Phase 7: Write Iteration Summary

After evaluation, write a summary of this iteration.

**Write to** `.autopilot/iteration-{ITERATION}/summary.md`:

```markdown
# Iteration {ITERATION} Summary

## Score: {overall_score}/100 (delta: {delta})
## Verdict: {verdict} ({stop_reason or "continuing"})

## Ideas Generated
- Total from /t:top-ideas: {count}
- Selected for this iteration: {count}
- Key ideas: {list}

## What Was Discussed
- Agreed features: {count from discussion-result}
- Disputed features: {count}
- Cut features: {count}

## What Was Built
- Beads completed: {count}
- Commits: {list of hashes}
- Key files: {list}

## Regression Review
- Regressions found: {regression_count}
- Test results: {passed}/{total} (previous: {previous_passed})
{if regressions: list each regression with cause and whether it was fixed}

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

- Mode: {MODE}
- Current iteration: {ITERATION}
- Latest score: {overall_score}/100
- Status: {running|completed}
- Stop reason: {reason or "in progress"}
- {if unlimited: "Action this iteration: {selected_actions}"}
- {if unlimited and focus_shift: "FOCUS SHIFT: strategy changed due to diminishing returns"}
- Last updated: {timestamp}
```

**Append to** `.autopilot/HISTORY.md` (create the file with a `# Autopilot Iteration History` heading if it does not yet exist). Add one block per iteration. Keep the top-level bullets scannable, but include a nested **Discussion** section that preserves the Proposer vs Challenger debate detail — this is the primary signal future iterations use to avoid re-arguing settled scope.

Parse `.autopilot/iteration-{ITERATION}/discussion-result.md` to populate the Discussion section. If that file is missing (discussion skipped/failed), emit `- Discussion: skipped ({reason})` instead of the nested block.

```markdown
## Iteration {ITERATION} — {YYYY-MM-DD HH:MM}
- Score: {overall_score}/100 (delta: {delta})
- Mode: {MODE}
- Actions: {selected_actions or "discuss → plan → build → review → evaluate"}
- Ideas selected: {count} ({first idea title}, ...)
- Beads closed: {count} ({first bead id}, ...)
- Commits: {short hashes, comma-separated}
- Regressions: {regression_count} ({fixed_count} fixed)
- Criteria met: {met_count}/{total_count}
- Verdict: {continue|stop} ({stop_reason or "in progress"})
- Discussion: {ROUND_COUNT} rounds, agreement {agreement_ratio}, exit: {convergence_reason}
  - Proposer [ADD]: {comma-separated ADD items from Proposer's final round, truncate each to ~60 chars}
  - Proposer [KEEP]: {KEEP items}
  - Proposer [MODIFY]: {MODIFY items with short note}
  - Challenger [CUT]: {CUT items from Challenger's final round}
  - Challenger [KEEP]: {KEEP items}
  - Challenger [MODIFY]: {MODIFY items with short note}
  - Agreed (built): {items in "Agreed Features" section}
  - Disputed: {items tagged [DISPUTED] with one-line Proposer-vs-Challenger rationale}
  - Cut: {items in "Explicitly Cut" section}
  - Concessions — Proposer: {list}; Challenger: {list}
```

Rules for the Discussion block:

1. Pull **final-round** tagged items only (not every round) — earlier rounds are intermediate state.
2. Quote item text verbatim from the agent messages; truncate with `…` after ~60 chars if needed.
3. Omit any sub-bullet whose list is empty (e.g., no `Proposer [MODIFY]:` line if the Proposer issued no MODIFY tags).
4. For `Disputed:`, use the format `{item} — P: {rationale}; C: {rationale}` so both stances survive.
5. Do NOT elide Concessions even if short — they are the highest-signal artifact for "what the agents gave up."

HISTORY.md is the durable, consolidated log that survives across sessions. It is committed to git (unlike the gitignored `STATUS.md`) and is the primary input the controller uses when **resuming** a session — see "Resume Flow" below.

---

## Iteration History Log

`.autopilot/HISTORY.md` is a single append-only file that captures a one-block summary for every completed iteration across every run of the loop. It exists for three reasons:

1. **Human scannability** — one file to read to understand the full arc of the project.
2. **Resume context** — when a session is resumed (via `/autopilot` detecting prior state), this file is read into Phase 1 and Phase 2 so agents do not re-propose already-completed work.
3. **Durability across session boundaries** — unlike `STATUS.md` (ephemeral, gitignored), HISTORY.md is committed and survives any session break.

### When it is written

- **End of Phase 7** of every iteration (see append block above).
- A single entry per iteration. If an iteration is re-run after crash recovery, the existing entry for that iteration must be **replaced**, not duplicated — match on the `## Iteration {ITERATION} —` heading.

### When it is read

- **Phase 1 (Idea Generation)** for iteration ≥ 2 — passed as `PRIOR_ITERATION_HISTORY` context so `/t:top-ideas` does not regenerate already-done work.
- **Phase 2 (Auto-Discussion)** — Proposer and Challenger receive it so they do not re-debate settled scope.
- **Resume Flow** — the controller reads it at startup to know which iteration to begin at and what was already accomplished.

### Format guarantees

- New iterations are **appended** at the bottom. The file is read top-to-bottom as a chronological log.
- The top-level bullets (Score through Verdict) stay ≤ 10 lines for scannability.
- The nested **Discussion** block may span additional lines — it preserves the Proposer/Challenger debate (tagged items, disputes, concessions) so future iterations can skip re-arguing settled scope. Omit empty sub-bullets to keep it tight. Full discussion detail still lives in `.autopilot/iteration-N/discussion-result.md`.
- Never rewrite prior entries — only append new ones (except for the crash-recovery re-run case above).

---

## Loop Termination: Final Report

When the loop stops (any reason), write `.autopilot/FINAL.md`:

```markdown
# Autopilot Final Report

## Configuration
- Idea: {IDEA}
- Mode: {MODE}
- Success criteria: {count} (auto-generated)
- Max iterations: {MAX_ITERATIONS or "unlimited"}
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

## Ideas Explored
{for each iteration, list the top ideas generated and which were selected}

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
| `/t:top-ideas` fails | Derive ideas from unmet criteria and original idea |
| `/t:discuss` fails | Use selected ideas directly as planning input |
| Discussion skill fails | Use selected ideas directly as planning input |
| Planning produces no beads | Skip build, go to evaluation (likely goals already met) |
| Building fails mid-execution | Proceed to regression review, then evaluation — low score triggers next iteration |
| Regression review fails | Log failure, proceed to evaluation (evaluator has its own regression detection) |
| Evaluation fails | Score 0, verdict "continue" — ensures another attempt |
| Agent Mail unavailable | Skills fall back to direct Task() prompts without thread persistence |
| `.autopilot/` directory missing | Re-create from config and continue |

### Resume Flow

Applies to BOTH (a) mid-iteration crash recovery and (b) user-initiated resume after `/autopilot` detects prior state in Step 0 and the user chose **Resume**.

1. **Re-hydrate configuration** — Read `.autopilot/config.md` to restore `IDEA`, `MODE`, `SUCCESS_CRITERIA`, `CONSTRAINTS`, and thresholds. Do NOT regenerate these — they were agreed at kickoff.
2. **Load prior iteration history** — Read `.autopilot/HISTORY.md`. Use it to derive:
   - `LAST_ITERATION` — the highest `## Iteration N —` heading found, or `0` if HISTORY.md is missing.
   - `PRIOR_ITERATION_HISTORY` — the full contents, passed as context into Phase 1 and Phase 2 so agents do not re-propose completed work.
3. **Detect partial iteration** — Look at `.autopilot/iteration-{LAST_ITERATION+1}/`:
   - If the directory exists but lacks `summary.md`, the prior session was killed mid-iteration. Resume from the next incomplete phase within that iteration (see phase → artifact mapping below).
   - If the directory does not exist (or `summary.md` is present for `LAST_ITERATION`), start a fresh iteration at `LAST_ITERATION + 1`.
4. **Phase → artifact mapping** for partial-iteration resume:

   | Phase | Artifact that signals completion |
   |-------|----------------------------------|
   | Phase 0 (unlimited) | `action-plan.md` |
   | Phase 1 | `top-ideas.md` AND `selected-ideas.md` |
   | Phase 2 | `discussion-result.md` |
   | Phase 3 (planning) | At least one open bead created post-`config.md` mtime, OR summary comment `planning complete` in the iteration directory |
   | Phase 4 (build) | Beads closed matching this iteration's plan, OR `build-errors.md` |
   | Phase 4.5 | `review-result.md` |
   | Phase 5 | `evaluation.json` |
   | Phase 6/7 | `summary.md` AND entry in HISTORY.md |

5. **Skip already-completed phases.** Do NOT re-run any phase whose artifact already exists — that would overwrite work and could double-count regressions or re-generate the same ideas.
6. **Update STATUS.md** at the start of the resumed phase: set `Status: running`, `Last updated: {now}`, and record the phase being entered.
7. **Pass PRIOR_ITERATION_HISTORY to Proposer/Challenger.** When the resumed run reaches Phase 2, the discussion context MUST include HISTORY.md contents under the header `## Prior Iteration History`. Proposer and Challenger are instructed not to re-propose ideas already marked done in HISTORY.md.

### Crash Recovery (special case of Resume Flow)

A crash is simply a Resume Flow that begins without a user prompt — invoked automatically when the controller restarts and finds `.autopilot/STATUS.md` with `Status: running`. The rules above apply unchanged.

---

## Available Skills for Phase Enhancement

The controller MAY invoke any of these global skills when the situation warrants it. Skills cost tokens and time — invoke them when the expected value is high, not mechanically every iteration.

| Phase | Skill | When to Use |
|-------|-------|-------------|
| Idea Generation | `brainstorming` | When `/t:top-ideas` produces stale or repetitive ideas |
| Idea Generation | `docs-seeker` | When the project needs external patterns or library research |
| Planning | `coding-standards` | When the plan involves unfamiliar patterns — check conventions first |
| Building | `test-driven-development` | When workers should write tests before implementation |
| Review (4.5) | `qa-sweep` | When code-reviewer finds many issues — run a full quality sweep |
| Review (4.5) | `security-review` | When changes touch auth, user input, APIs, or sensitive data |
| Evaluation | `t:fresh-eyes` | When evaluator scores seem inconsistent — fresh perspective helps |
| Debugging | `t:rootfix` | When tests fail repeatedly — diagnose root cause instead of patching |
| Polish | `t:polish` | When score > 80 but criteria remain unmet — focus on fit and finish |

**In unlimited mode**, Phase 0 (AI-driven action selection) should consider invoking these skills as part of its action plan. For example, if the action plan selects `security-review` as an action type, invoke the `security-review` skill directly rather than just building security features.

---

## State File Reference

| File | Written By | Read By | Purpose |
|------|-----------|---------|---------|
| `.autopilot/config.md` | /autopilot command | All phases | Idea, criteria, thresholds, mode |
| `.autopilot/STATUS.md` | Controller (this) | Crash recovery | Current state |
| `.autopilot/iteration-N/action-plan.md` | Phase 0 (unlimited mode) | Phase 1, Phase 2 | AI-driven action selection |
| `.autopilot/iteration-N/focus-shift.md` | Phase 6 (unlimited mode) | Phase 0 next iteration | Diminishing returns strategy shift |
| `.autopilot/iteration-N/top-ideas.md` | Phase 1 (this) | Phase 2 | Generated ideas |
| `.autopilot/iteration-N/selected-ideas.md` | Phase 1 (this) | Phase 2 | Picked ideas |
| `.autopilot/iteration-N/discussion-result.md` | /t:discuss or discussion skill | Planning phase | What to build |
| `.autopilot/iteration-N/review-result.md` | Phase 4.5 (code-reviewer) | Evaluator, human review | Regression review findings |
| `.autopilot/iteration-N/evaluation.json` | Evaluator agent | Controller, next iteration | Scores and verdict |
| `.autopilot/iteration-N/summary.md` | Controller (this) | Human review | Iteration recap |
| `.autopilot/HISTORY.md` | Controller (this, Phase 7 append) | Resume Flow, Phase 1, Phase 2, human review | Consolidated, committed cross-iteration log. Primary input for resume and for agent awareness of what's already been done. |
| `.autopilot/archive/{timestamp}/` | `/autopilot` Step 0 (on "Start fresh") | Human review only | Snapshot of prior session state when user opts for a fresh start |
| `.autopilot/FINAL.md` | Controller (this) | Human review | Final report |
