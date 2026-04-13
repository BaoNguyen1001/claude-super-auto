# /autopilot — Fully Autonomous Idea-to-Implementation Loop

User provides ONE initial request. Everything else is autonomous. No further questions asked.

## Step 1: Accept the Initial Request

If the user provided arguments with the command, use those as `IDEA`.

If no arguments were provided, ask the user ONE question:

> **What do you want to build or improve?**
> Describe your idea in a few sentences.

Wait for response. Store as `IDEA`. This is the ONLY question you ask. Do NOT ask for success criteria, constraints, iteration settings, or anything else.

## Step 2: Auto-Generate Configuration

Autonomously analyze the `IDEA` and the current codebase to generate:

1. **Success Criteria** — Derive 3-7 concrete, verifiable criteria from the idea. Read the codebase to understand what already exists and what the idea implies.
2. **Constraints** — Infer from the codebase (language, framework, existing patterns). Do NOT ask the user.
3. **Thresholds** — Use defaults: `goal_threshold: 85`, `delta_threshold: 5`, `max_iterations: 5`

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

## Success Criteria (auto-generated)
{SUCCESS_CRITERIA as bulleted list}

## Constraints (auto-inferred)
{CONSTRAINTS}

## Thresholds
- goal_threshold: 85
- delta_threshold: 5
- max_iterations: 5
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
  AUTOPILOT ENGAGED
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Idea: {IDEA (truncated to 80 chars)}
  Auto-generated criteria: {count}
  Max iterations: 5

  The system will now FULLY AUTONOMOUSLY:
  1. Generate top ideas via /t:top-ideas
  2. Auto-pick and discuss via /t:discuss
  3. Plan and decompose into tasks
  4. Build with TDD
  5. Evaluate and loop

  You can walk away. Check .autopilot/STATUS.md
  for progress, or .autopilot/FINAL.md when complete.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## Step 5: Invoke the Loop Controller

Hand off to the autopilot-loop skill for fully autonomous execution:

```
Skill(skill="autopilot-loop")
```

From this point forward, no user interaction occurs. The controller manages all phases, iterations, and stop conditions autonomously.

When the controller finishes, it writes `.autopilot/FINAL.md` with the complete results.
