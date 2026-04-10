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

You are running the **risk-scorer** skill. Evaluate the functional areas of a story and rank them by risk. Do all the work directly — no sub-agents.

## Step 1 — Get the ticket key

If the user has not provided a Jira ticket key, ask for one now.

## Step 2 — Check prerequisites

Look for `qa-output/story-pipeline/<KEY>/02-enriched-ac.md`.

If it does not exist:
> "I need enriched ACs before I can score risk. Run `/ac-enricher [KEY]` first."

Read the enriched ACs file now.

## Step 3 — Score risk

Group the scenarios into functional areas or theme clusters (e.g. "cancellation flow", "notification handling", "error states"). These groups are your units of analysis.

For each functional area, assign two scores from 1 to 5:

**Likelihood of failure (1–5)**
- 1 = Very unlikely (simple, well-understood, previously tested)
- 3 = Possible (some complexity, new code, or integration point)
- 5 = Very likely (high complexity, new feature, external dependency, historically buggy area)

**Business impact (1–5)**
- 1 = Minimal (cosmetic, no data loss, easy workaround)
- 3 = Moderate (degraded experience, workaround available)
- 5 = Severe (data loss, security issue, blocks core user journey, regulatory risk)

**Risk score** = Likelihood × Impact

**Classification**:
- **HIGH**: Score ≥ 15
- **MEDIUM**: Score 8–14
- **LOW**: Score ≤ 7

## Step 4 — Save the risk matrix

Save to `qa-output/story-pipeline/<KEY>/03-risk-matrix.md`:

```markdown
# Risk Matrix — [TICKET-KEY]: [Story Title]

## Risk Summary
- HIGH risk areas: X
- MEDIUM risk areas: X
- LOW risk areas: X

## Risk Matrix

| # | Functional Area | Likelihood (1–5) | Impact (1–5) | Risk Score | Level |
|---|---|---|---|---|---|
| 1 | [Area name] | 5 | 5 | 25 | HIGH |

## Recommended Test Execution Order

1. **[Area name]** — HIGH (Score: 25) — [One sentence on why this area is critical]

## Risk Rationale

### [Area name] — HIGH
[2–3 sentences explaining likelihood of failure and business impact]
```

## Step 5 — Present the results

Show the risk matrix table and recommended test execution order. Highlight:
- How many HIGH risk areas were found
- The top 2–3 riskiest areas and why they scored high

Then suggest next step:
> "Risk matrix is ready. Next step: run `/testcase-builder [KEY]` to generate structured test cases ordered by risk priority."
