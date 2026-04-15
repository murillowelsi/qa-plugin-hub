---
name: ac-enricher
description: >
  Rewrites raw Jira acceptance criteria into structured Given/When/Then BDD
  scenarios, adding at least one negative path and one edge case per AC.
  Produces enriched scenarios that the whole team — developers, testers, and
  product owner — can read and validate.
  Use this skill after DoR passes, when the user wants to improve AC quality,
  asks "can you write BDD scenarios for this?", "rewrite the ACs", "add edge
  cases to the acceptance criteria", "turn these ACs into Given/When/Then",
  "enrich the acceptance criteria", or wants to prepare the story for test design.
  Always use this skill — not manual rewriting — whenever BDD scenarios or enriched ACs are needed.
allowed-tools: Read, Write, mcp__*__getJiraIssue, mcp__*__getConfluencePage, mcp__*__addCommentToJiraIssue
---

# AC Enricher

You are running the **ac-enricher** skill. Transform raw acceptance criteria into rich, structured BDD scenarios. Do all the work directly — no sub-agents.

## Step 1 — Get the ticket key

If the user has not provided a Jira ticket key, ask for one now.

## Step 2 — Check prerequisites

Look for `qa-output/issue-pipeline/<KEY>/dor-verdict.json`.

- **If verdict is `"BLOCK"`**: stop here and tell the user:
  > "⚠️ Story [KEY] was blocked at the DoR gate. Enriching ACs now would likely be wasted work — the story may be rewritten after the blocking issues are fixed. Run `/issue-refiner [KEY]` to fix the story first, then re-run `/dor-gatekeeper [KEY]` to clear it, and come back here."
  Do not proceed without explicit user confirmation (e.g. "proceed anyway", "I know, continue").

- **If the file does not exist**: stop and tell the user:
  > "ℹ️ The DoR gate hasn't been run for [KEY] yet. Run `/dor-gatekeeper [KEY]` first to confirm the story is ready before enriching its ACs. You can also run `/issue-pipeline [KEY]` to do the full sequence automatically."
  Do not proceed without explicit user confirmation (e.g. "skip it", "proceed anyway").

## Step 3 — Fetch the story

Fetch the full Jira story using the MCP tool to get the original acceptance criteria. Also read the analysis report at `qa-output/issue-pipeline/<KEY>/01-analysis.md` if it exists, for context on known gaps.

## Step 4 — Enrich the ACs

For each original acceptance criterion:

1. **Rewrite as Given/When/Then** — each scenario must be standalone and self-explanatory
2. **Add at least one negative path** — what happens when input is wrong, missing, or invalid?
3. **Add at least one edge case** — boundary values, empty states, concurrent actions, maximum limits

Keep language precise but non-technical. No implementation details. Readable by a product owner, verifiable by a developer.

**Scenario format**:

### Scenario: [Short descriptive title]
**Type**: Positive | Negative | Edge Case
```gherkin
Given [initial context or precondition]
When [the action the user or system takes]
Then [the observable outcome]
And [additional outcome, if needed]
```

Group scenarios under the original AC they derive from.

## Step 5 — Save output

Save enriched ACs to `qa-output/issue-pipeline/<KEY>/02-enriched-ac.md`.

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
