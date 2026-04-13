# /autopilot — Fully Autonomous Idea-to-Implementation Loop

User provides ONE initial request. Everything else is autonomous. No further questions asked.

## Step 1: Accept the Initial Request

If the user provided arguments with the command, use those as `IDEA`.

If no arguments were provided, ask the user ONE question:

> **What do you want to build or improve?**
> Describe your idea in a few sentences.

Wait for response. Store as `IDEA`. This is the ONLY question you ask. Do NOT ask for success criteria, constraints, iteration settings, or anything else.

## Step 1.5: Detect Mode

Check if the user specified a mode in their arguments (e.g., `--unlimited`, `unlimited`, `--mode unlimited`).

If no mode was specified, ask ONE follow-up question:

> **Which mode?**
> - **A) Bounded** (default) — runs up to 5 iterations, stops when goals met or diminishing returns
> - **B) Unlimited** — runs indefinitely until goals met (score >= 95%) or you interrupt the session. Fully autonomous — no questions, no stops. The agent decides what to do each iteration.

Store the choice as `MODE` (`bounded` or `unlimited`).

If the user doesn't answer or says "default", use `bounded`.

## Step 2: Auto-Generate Configuration

Autonomously analyze the `IDEA` and the current codebase to generate:

1. **Success Criteria** — Derive 3-7 concrete, verifiable criteria from the idea. Read the codebase to understand what already exists and what the idea implies.
2. **Constraints** — Infer from the codebase (language, framework, existing patterns). Do NOT ask the user.
3. **Thresholds** — Depend on mode:
   - **Bounded**: `goal_threshold: 85`, `delta_threshold: 5`, `max_iterations: 5`
   - **Unlimited**: `goal_threshold: 95`, `delta_threshold: 3`, `max_iterations: null` (ignored)

## Step 3: Initialize State

Create the `.autopilot/` directory and write the configuration:

```bash
mkdir -p .autopilot
```

Write `.autopilot/config.md`:

```markdown
# Autopilot Configuration

## Idea
{IDEA}

## Mode
{MODE}

## Success Criteria (auto-generated)
{SUCCESS_CRITERIA as bulleted list}

## Constraints (auto-inferred)
{CONSTRAINTS}

## Thresholds
- mode: {MODE}
- goal_threshold: {85 if bounded, 95 if unlimited}
- delta_threshold: {5 if bounded, 3 if unlimited}
- max_iterations: {5 if bounded, null if unlimited}
```

Write `.autopilot/STATUS.md`:

```markdown
# Autopilot Status

- Current iteration: 0
- Latest score: N/A
- Status: initializing
- Stop reason: N/A
- Last updated: {current timestamp}
```

## Step 4: Confirm and Launch

Display to the user:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  AUTOPILOT ENGAGED ({MODE} mode)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Idea: {IDEA (truncated to 80 chars)}
  Auto-generated criteria: {count}
  Mode: {MODE}
  {if bounded: "Max iterations: 5 | Goal: >= 85%"}
  {if unlimited: "Iterations: UNLIMITED | Goal: >= 95% | Stops only when goals met or session interrupted"}

  The system will now FULLY AUTONOMOUSLY:
  1. {if bounded: "Generate top ideas via /t:top-ideas"}
     {if unlimited: "Analyze project state and decide what to do (AI-driven)"}
  2. Auto-pick and discuss via adversarial debate
  3. Plan and decompose into tasks
  4. Build with TDD
  5. Review for regressions
  6. Evaluate and loop

  {if unlimited: "UNLIMITED MODE: The agent will NOT ask you anything."}
  {if unlimited: "It will autonomously decide each iteration's focus:"}
  {if unlimited: "build, test, review, refactor, fix bugs, upgrade, etc."}
  {if unlimited: "Break the session to stop."}

  Check .autopilot/STATUS.md for progress,
  or .autopilot/FINAL.md when complete.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## Step 5: Invoke the Loop Controller

Hand off to the autopilot-loop skill for fully autonomous execution:

```
Skill(skill="autopilot-loop")
```

From this point forward, no user interaction occurs. The controller manages all phases, iterations, and stop conditions autonomously.

When the controller finishes, it writes `.autopilot/FINAL.md` with the complete results.
