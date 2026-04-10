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

You are running the **ac-enricher** skill. Your job is to transform raw acceptance criteria into rich, structured BDD scenarios that cover the main path, negative paths, and edge cases.

## Step 1 — Get the ticket key

If the user has not provided a Jira ticket key, ask for one now.

## Step 2 — Check prerequisites

**DoR verdict**: Look for `qa-output/story-pipeline/<KEY>/dor-verdict.json`.

- If the file exists and verdict is `"BLOCK"`: warn the user that the story was blocked at the DoR gate, but offer to proceed anyway if they confirm.
- If the file does not exist: inform the user that the DoR gate has not been run, and suggest running `/dor-gatekeeper [KEY]` first — but offer to proceed directly if they prefer.

## Step 3 — Spawn the ac-enricher agent

Delegate to the **ac-enricher** agent, passing:
- The ticket key
- The path to the analysis report (if it exists): `qa-output/story-pipeline/<KEY>/01-analysis.md`

The agent will:
- Fetch the original ACs from Jira
- Rewrite each as a Given/When/Then scenario
- Add at least one negative path and one edge case per AC
- Save enriched scenarios to `qa-output/story-pipeline/<KEY>/02-enriched-ac.md`

## Step 4 — Present the results

Display a summary of what was generated:
- How many original ACs were found
- How many scenarios were created in total (positive / negative / edge case)
- Show a preview of the first 2–3 scenarios

Then tell the user where the full output was saved.

## Step 5 — Suggest next step

> "Enriched ACs are ready. Next step: run `/risk-scorer [KEY]` to score each functional area by likelihood of failure and business impact, so we know where to focus testing."
