# /autopilot — Autonomous Idea-to-Implementation Loop

Interactive kickoff followed by fully autonomous execution. No human intervention after the kickoff questions.

**IMPORTANT**: Ask each question ONE AT A TIME. Wait for the user's response before asking the next question. Do not bundle multiple questions into a single message.

## Step 1: Gather the Idea

Ask the user ONLY this question:

> **What do you want to build?**
> Describe your idea in 1-3 sentences. Be specific about what the end result should be.

Wait for response. Store as `IDEA`. Do NOT ask the next question until the user has answered.

## Step 2: Gather Success Criteria

Ask the user ONLY this question:

> **How will we know it's done?**
> List 3-7 concrete success criteria. Each should be verifiable (something we can check in the code/output).
>
> Example:
> - CLI accepts a markdown file as input
> - Outputs a formatted PDF
> - Supports syntax highlighting in code blocks

Wait for response. Store as `SUCCESS_CRITERIA`. Do NOT ask the next question until the user has answered.

## Step 3: Gather Constraints (optional)

Ask the user ONLY this question:

> **Any constraints or preferences?** (optional — press enter to skip)
>
> Examples: "Use TypeScript", "No external dependencies", "Must work offline"

Wait for response. Store as `CONSTRAINTS`. Do NOT ask the next question until the user has answered.

## Step 4: Gather Iteration Settings (optional)

Ask the user ONLY this question:

> **Iteration settings** (defaults shown — press enter to accept all):
> - Max iterations: 5
> - Goal score threshold: 85/100
> - Diminishing returns threshold: 5 points

Wait for response. Parse any overrides. Store as `MAX_ITERATIONS`, `GOAL_THRESHOLD`, `DELTA_THRESHOLD`.

## Step 5: Initialize State

Create the `.autopilot/` directory and write the configuration:

```bash
mkdir -p .autopilot
```

Write `.autopilot/config.md`:

```markdown
# Autopilot Configuration

## Idea
{IDEA}

## Success Criteria
{SUCCESS_CRITERIA as bulleted list}

## Constraints
{CONSTRAINTS or "None specified"}

## Thresholds
- goal_threshold: {GOAL_THRESHOLD or 85}
- delta_threshold: {DELTA_THRESHOLD or 5}
- max_iterations: {MAX_ITERATIONS or 5}
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

Ensure `.autopilot/.gitignore` exists (should have been created during project setup).

## Step 6: Confirm and Launch

Display to the user:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  AUTOPILOT ENGAGED
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Idea: {IDEA (truncated to 80 chars)}
  Criteria: {count} defined
  Max iterations: {MAX_ITERATIONS}
  Goal threshold: {GOAL_THRESHOLD}/100

  The system will now autonomously:
  1. Discuss scope via /t:discuss (Proposer vs Challenger)
  2. Plan and decompose into tasks
  3. Build with TDD
  4. Evaluate progress
  5. Loop until done

  You can walk away. Check .autopilot/STATUS.md
  for progress, or .autopilot/FINAL.md when complete.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## Step 7: Invoke the Loop Controller

Hand off to the autopilot-loop skill for fully autonomous execution:

```
Skill(skill="autopilot-loop")
```

From this point forward, no user interaction occurs. The controller manages all phases, iterations, and stop conditions autonomously.

When the controller finishes, it writes `.autopilot/FINAL.md` with the complete results.
