---
name: discussion
description: Orchestrates adversarial debate between Proposer and Challenger agents. Mediates rounds, detects convergence, and synthesizes a discussion result. Invoked by the autopilot-loop controller.
model: opus
---

# Discussion Skill: Adversarial Debate Protocol

This skill mediates a structured debate between two sub-agents (Proposer and Challenger) to refine an idea into a concrete, debated feature list.

## Inputs

The caller (autopilot-loop controller) provides:

- `ITERATION` — current loop iteration number (1, 2, 3, ...)
- `IDEA` — the original user idea
- `SUCCESS_CRITERIA` — list of criteria from `.autopilot/config.md`
- `CONSTRAINTS` — optional constraints from config
- `PRIOR_EVALUATION` — (iteration 2+ only) previous evaluation.json results
- `COMPLETED_ITEMS` — (iteration 2+ only) items already built
- `REMAINING_ITEMS` — (iteration 2+ only) items not yet built

## Output

Writes `.autopilot/iteration-{ITERATION}/discussion-result.md`

---

## Phase 1: Post Seed Message

Create the Agent Mail thread and post the seed message that frames the debate.

### Iteration 1 Seed Template

```
send_message(
  to=["Proposer", "Challenger"],
  thread_id="discussion-{ITERATION}",
  subject="[Seed] Discussion for Iteration {ITERATION}",
  body_md="""
## Idea
{IDEA}

## Success Criteria
{SUCCESS_CRITERIA as bulleted list}

## Constraints
{CONSTRAINTS or "None specified"}

## Instructions
Proposer: Expand this idea into a concrete feature list. What should we build?
Challenger: Evaluate the Proposer's expansion. What should we cut, simplify, or keep?

Use the structured message format (KEEP/ADD/CUT/MODIFY tags).
"""
)
```

### Iteration 2+ Seed Template

```
send_message(
  to=["Proposer", "Challenger"],
  thread_id="discussion-{ITERATION}",
  subject="[Seed] Discussion for Iteration {ITERATION}",
  body_md="""
## Idea
{IDEA}

## Success Criteria
{SUCCESS_CRITERIA as bulleted list}

## Prior Iteration Results
- **Score**: {PRIOR_EVALUATION.overall_score}/100
- **Delta from previous**: {PRIOR_EVALUATION.delta}
- **Evaluator summary**: {PRIOR_EVALUATION.summary}

## Criteria Status
{for each criterion in PRIOR_EVALUATION.criteria_checklist}
- {'MET' if met else 'UNMET'}: {criterion}

## Already Completed
{COMPLETED_ITEMS as bulleted list}

## Not Yet Built
{REMAINING_ITEMS as bulleted list}

## Directive
Focus discussion on addressing UNMET criteria and evaluator feedback.
Do NOT re-debate items that are already completed and working.
Proposer: What should we add or improve to meet remaining criteria?
Challenger: Is each proposal the simplest way to address the gap?
"""
)
```

---

## Phase 2: Run Debate Rounds

Execute up to 3 rounds. Each round spawns Proposer then Challenger sequentially.

```
for ROUND in 1, 2, 3:

    # Step 1: Spawn Proposer
    proposer_result = Task(
        subagent_type="proposer",
        prompt="""
        You are the Proposer in round {ROUND} of discussion iteration {ITERATION}.
        
        Read the Agent Mail thread: fetch_inbox() for thread "discussion-{ITERATION}"
        Read all prior messages, then post your response following your agent instructions.
        
        Thread ID: discussion-{ITERATION}
        Round: {ROUND}
        """
    )

    # Step 2: Spawn Challenger
    challenger_result = Task(
        subagent_type="challenger",
        prompt="""
        You are the Challenger in round {ROUND} of discussion iteration {ITERATION}.
        
        Read the Agent Mail thread: fetch_inbox() for thread "discussion-{ITERATION}"
        Read all prior messages including the Proposer's round {ROUND} message, then post your response.
        
        Thread ID: discussion-{ITERATION}
        Round: {ROUND}
        """
    )

    # Step 3: Check convergence
    converged = check_convergence(ROUND)
    if converged:
        break
```

### Format Violation Handling

If a Proposer or Challenger message does NOT contain the required `## Items` section with tagged items:

1. Retry ONCE with a correction prompt:
```
Task(
    subagent_type="{agent}",
    prompt="""
    Your previous message was rejected — it did not follow the required format.
    
    You MUST include a ## Items section with tagged items: [KEEP], [ADD], [CUT], [MODIFY].
    You MUST include ## Confidence, ## Concessions, and ## Open Questions sections.
    
    Re-read the thread and post a correctly formatted response.
    Thread ID: discussion-{ITERATION}
    Round: {ROUND}
    """
)
```

2. If the retry also fails, use the malformed message as-is and proceed — do not loop indefinitely.

---

## Phase 3: Convergence Detection

After each round, parse both agents' messages and compute convergence metrics.

### Parsing Algorithm

1. Extract the `## Items` section from both Proposer and Challenger messages
2. Parse each line matching the pattern `- \[(KEEP|ADD|CUT|MODIFY)\] (.+)`
3. Normalize item text: lowercase, strip leading/trailing whitespace, remove parenthetical rationale
4. Build two sets: `proposer_items` and `challenger_items`, each as `{tag, normalized_text}`

### Convergence Metrics

```
# Items both sides tagged as [KEEP]
agreed = proposer_items.where(tag=KEEP) ∩ challenger_items.where(tag=KEEP)

# All unique items across both messages (by normalized text)
total_unique = union(proposer_items, challenger_items) by normalized_text

# Items where one says [ADD] and the other says [CUT]
disputes = items where proposer.tag=ADD and challenger.tag=CUT
           OR proposer.tag=KEEP and challenger.tag=CUT

# Metrics
agreement_ratio = |agreed| / |total_unique|
open_questions_count = |disputes|
```

### Stop Conditions (ANY triggers convergence)

| Condition | Threshold | Meaning |
|-----------|-----------|---------|
| High agreement | `agreement_ratio >= 0.8` | Supermajority of items are agreed upon |
| No disputes | `open_questions_count == 0` | No items have conflicting tags |
| Round cap | `ROUND == 3` | Hard cap reached, force-synthesize |

### Fuzzy Item Matching

Items from different agents may use slightly different wording for the same concept. To match them:

1. Normalize: lowercase, remove stop words (a, an, the, with, for, to, of), strip punctuation
2. Compare using Jaccard similarity on word sets: `|A ∩ B| / |A ∪ B|`
3. Threshold: similarity >= 0.6 means same item

If Jaccard matching is insufficient, use an LLM call to canonicalize item names:
```
Task(
    subagent_type="general-purpose",
    model="haiku",
    prompt="Given these two lists of items, identify which items refer to the same concept. Output pairs."
)
```

---

## Phase 4: Synthesis

When the debate ends (convergence or round cap), synthesize the final discussion result.

### Synthesis Rules

Process items from the **final round** messages:

| Proposer Tag | Challenger Tag | Result |
|-------------|---------------|--------|
| [KEEP] | [KEEP] | **Included** as-is |
| [ADD] | [KEEP] | **Included** (Challenger agreed) |
| [KEEP] | [ADD] | **Included** (both agree) |
| [ADD] | — (not mentioned) | **Included** (unchallenged) |
| — (not mentioned) | [CUT] | **Dropped** (only Challenger mentioned, to cut) |
| [CUT] | [CUT] | **Dropped** (both agree to remove) |
| [ADD] | [CUT] | **[DISPUTED]** — include with tag for planning to resolve |
| [MODIFY] | [KEEP] | **Included** with modification (Proposer's version) |
| [KEEP] | [MODIFY] | **Included** with modification (Challenger's version) |
| [MODIFY] | [MODIFY] | **Included** with last-stated version |

### Output Document

Write to `.autopilot/iteration-{ITERATION}/discussion-result.md`:

```markdown
# Discussion Result — Iteration {ITERATION}

## Convergence
- Rounds completed: {ROUND_COUNT}
- Agreement ratio: {agreement_ratio}
- Open disputes: {open_questions_count}
- Exit reason: {convergence_reason}

## Agreed Features (build these)
- <item 1>
- <item 2>
- ...

## Disputed Features (planning phase decides)
- [DISPUTED] <item> — Proposer: {rationale}. Challenger: {rationale}.
- ...

## Explicitly Cut (do NOT build)
- <item> — cut because: {reason}
- ...

## Key Concessions
- Proposer conceded: {list}
- Challenger conceded: {list}

## Recommendations for Planning
<Brief synthesis of what should be built and in what order, based on the debate>
```

---

## Error Recovery

| Failure | Action |
|---------|--------|
| Agent Mail unavailable | Fall back to direct Task() prompt/response without thread persistence |
| Agent produces no output | Retry once, then proceed with other agent's items only |
| Both agents fail | Return seed message items as discussion result (no debate) |
| Convergence never reached | Force-synthesize after round 3 with [DISPUTED] tags on conflicts |

---

## Quick Reference

| Step | Tool | Purpose |
|------|------|---------|
| Post seed | `send_message()` | Frame the debate |
| Spawn agents | `Task(subagent_type="proposer/challenger")` | Run each round |
| Read messages | `fetch_inbox()`, `summarize_thread()` | Parse positions |
| Fuzzy match | Haiku Task or manual Jaccard | Canonicalize items |
| Write result | `Write()` | Save discussion-result.md |
