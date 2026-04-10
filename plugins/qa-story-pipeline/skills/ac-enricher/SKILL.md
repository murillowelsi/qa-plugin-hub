---
name: ac-enricher
description: >
  Rewrites raw Jira acceptance criteria into structured Given/When/Then BDD
  scenarios, adding at least one negative path and one edge case per AC.
  Produces enriched scenarios that the whole team — developers, testers, and
  product owner — can read and validate.
  Use this skill after DoR passes, when the user wants to improve AC quality,
  asks "can you write BDD scenarios for this?", "rewrite the ACs", "add edge
  cases to the acceptance criteria", "turn these ACs into Given/When/Then", or
  wants to prepare the story for test design.
---

# AC Enricher

You are running the **ac-enricher** skill. Transform raw acceptance criteria into rich, structured BDD scenarios. Do all the work directly — no sub-agents.

## Step 1 — Get the ticket key

If the user has not provided a Jira ticket key, ask for one now.

## Step 2 — Check prerequisites

Look for `qa-output/story-pipeline/<KEY>/dor-verdict.json`.

- If verdict is `"BLOCK"`: warn the user the story was blocked at DoR, but offer to proceed if they confirm.
- If the file does not exist: inform the user the DoR gate hasn't been run, suggest `/dor-gatekeeper [KEY]` first — but offer to proceed directly if they prefer.

## Step 3 — Fetch the story

Fetch the full Jira story using the MCP tool to get the original acceptance criteria. Also read the analysis report at `qa-output/story-pipeline/<KEY>/01-analysis.md` if it exists, for context on known gaps.

## Step 4 — Enrich the ACs

For each original acceptance criterion:

1. **Rewrite as Given/When/Then** — each scenario must be standalone and self-explanatory
2. **Add at least one negative path** — what happens when input is wrong, missing, or invalid?
3. **Add at least one edge case** — boundary values, empty states, concurrent actions, maximum limits

Keep language precise but non-technical. No implementation details. Readable by a product owner, verifiable by a developer.

**Scenario format**:
```
### Scenario: [Short descriptive title]
**Type**: Positive | Negative | Edge Case
**Given** [initial context or precondition]
**When** [the action the user or system takes]
**Then** [the observable outcome]
**And** [additional outcome, if needed]
```

Group scenarios under the original AC they derive from.

## Step 5 — Save output

Save enriched ACs to `qa-output/story-pipeline/<KEY>/02-enriched-ac.md`.

Structure:
- Header with ticket key and story title
- Each original AC as a section heading
- Enriched scenarios nested under it

## Step 6 — Present the results

Display:
- How many original ACs were found
- Total scenarios created (positive / negative / edge case breakdown)
- Preview of the first 2–3 scenarios
- Where the full output was saved

Then suggest next step:
> "Enriched ACs are ready. Next step: run `/risk-scorer [KEY]` to score each functional area by likelihood of failure and business impact."
