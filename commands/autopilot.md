# /autopilot — Fully Autonomous Idea-to-Implementation Loop

User provides ONE initial request. Everything else is autonomous. No further questions asked.

## Step 0: Detect Prior Session (Resume vs Fresh)

Before asking anything, check whether an earlier autopilot session exists that the user may want to resume.

1. **Check for prior state.** Read `.autopilot/STATUS.md` if it exists. Parse the `Status:` field.

2. **Classify the prior state:**

   | Status value | Meaning | Action |
   |--------------|---------|--------|
   | File missing | No prior session | Proceed to Step 1 (fresh kickoff) |
   | `initializing` | Session started but Phase 1 never ran | Treat as resumable but offer fresh |
   | `running` | Loop mid-flight when session ended | Resumable |
   | `completed` | Loop finished normally | Treat as archive-and-restart |

3. **If resumable, ask the user ONE question:**

   > **A prior autopilot session was detected.**
   > - Mode: `{MODE}`
   > - Last iteration: `{N}`
   > - Latest score: `{score}/100`
   > - Status: `{Status}` — `{Stop reason if any}`
   >
   > **What would you like to do?**
   > - **A) Resume** — continue from iteration `{N+1}` (or next incomplete phase of iteration `{N}`), reusing the existing config and idea.
   > - **B) Start fresh** — archive the prior session to `.autopilot/archive/{timestamp}/` and begin a new loop.

4. **On Resume:**
   - **Do NOT** ask for an idea, mode, or any kickoff inputs. They already live in `.autopilot/config.md`.
   - Update `.autopilot/STATUS.md`: set `Status: running` and `Last updated: {now}`.
   - Display a brief resume banner:
     ```
     ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
       AUTOPILOT RESUMED ({MODE} mode)
     ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
       Last iteration: {N} (score: {score}/100)
       Resuming from: {next phase or iteration N+1}
     ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
     ```
   - Skip to **Step 5** (invoke the loop controller). The controller's Resume Flow handles the rest.

5. **On Start fresh:**
   - Archive existing state. Move every child of `.autopilot/` EXCEPT `config.md.template`, `.gitignore`, and any existing `archive/` directory into `.autopilot/archive/{YYYY-MM-DD-HH-MM}/`:
     ```bash
     ARCHIVE_DIR=".autopilot/archive/$(date +%Y-%m-%d-%H-%M)"
     mkdir -p "$ARCHIVE_DIR"
     find .autopilot -mindepth 1 -maxdepth 1 \
       ! -name config.md.template \
       ! -name .gitignore \
       ! -name archive \
       -exec mv {} "$ARCHIVE_DIR/" \;
     ```
   - Proceed to Step 1 (fresh kickoff) as if no prior session existed.

6. **On missing STATUS.md:** Proceed to Step 1 — nothing to resume.

Rationale: Per D1 (fully autonomous) and D5 (interactive kickoff), this Step 0 prompt is the ONLY new user-facing question added to the command. After the resume/fresh decision, the loop stays fully autonomous.

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
