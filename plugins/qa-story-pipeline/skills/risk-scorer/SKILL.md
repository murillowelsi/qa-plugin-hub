---
name: risk-scorer
description: >
  Scores each functional area of a Jira story by likelihood of failure ×
  business impact using ISTQB risk-based testing principles. Produces a ranked
  risk matrix that guides test prioritization — HIGH risk areas get tested first.
  Use this skill after ac-enricher, when the user needs to prioritize testing
  effort, asks "where should we focus testing?", "what are the riskiest areas?",
  "score the risk", "risk assessment", or wants to know which parts of the story
  are most likely to fail or have the highest business impact if they do.
---

# Risk Scorer

You are running the **risk-scorer** skill. Your job is to evaluate the functional areas of a story and rank them by risk so the team knows where to focus testing effort.

## Step 1 — Get the ticket key

If the user has not provided a Jira ticket key, ask for one now.

## Step 2 — Check prerequisites

Look for `qa-output/story-pipeline/<KEY>/02-enriched-ac.md`.

If it does not exist, the ACs have not been enriched yet. Tell the user:
> "I need enriched ACs before I can score risk. Run `/ac-enricher [KEY]` first."

If the enriched ACs file exists, proceed.

## Step 3 — Spawn the risk-scorer agent

Delegate to the **risk-scorer** agent, passing:
- The ticket key
- The path to enriched ACs: `qa-output/story-pipeline/<KEY>/02-enriched-ac.md`

The agent will:
- Group scenarios into functional areas
- Score each area: Likelihood (1–5) × Impact (1–5) = Risk Score
- Classify areas as HIGH (≥15), MEDIUM (8–14), or LOW (≤7)
- Rank areas by risk score
- Save the risk matrix to `qa-output/story-pipeline/<KEY>/03-risk-matrix.md`

## Step 4 — Present the results

Show the risk matrix table and the recommended test execution order. Highlight:
- How many HIGH risk areas were found (these need most attention)
- The top 2–3 riskiest areas and why they scored high

## Step 5 — Suggest next step

> "Risk matrix is ready. Next step: run `/testcase-builder [KEY]` to generate structured test cases ordered by risk priority."
