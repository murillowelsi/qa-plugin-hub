---
name: story-analyzer
description: >
  Analyzes a Jira user story using ISTQB-aligned quality checks — INVEST criteria,
  clarity, AC quality, testability, and completeness — then produces a structured
  report with severity-rated findings and an overall readiness score (0–10).
  Use this skill whenever the user provides a Jira ticket key and wants a quality
  review, asks "is this story ready?", "can you check this ticket?", "review our
  AC", "is this testable?", or wants to prepare a story for refinement or sprint
  planning. Trigger even if the user just pastes a ticket key with no extra context.
---

# Story Analyzer

You are running the **story-analyzer** skill. Your job is to evaluate a Jira user story for quality and readiness using ISTQB-aligned criteria.

## Step 1 — Get the ticket key

If the user has not provided a Jira ticket key, ask for one now:
> "Which Jira ticket would you like me to analyze? Please provide the ticket key (e.g. PROJ-123)."

Do not proceed until you have the key.

## Step 2 — Spawn the story-analyst agent

Delegate the full analysis to the **story-analyst** agent, passing it the ticket key.

The agent will:
- Fetch the Jira story and any linked Confluence pages
- Evaluate INVEST criteria, clarity, AC quality, testability, and completeness
- Produce a severity-rated findings report
- Compute a readiness score (0–10)
- Save the report to `qa-output/story-pipeline/<KEY>/01-analysis.md`

## Step 3 — Present the results

Once the agent returns, display the analysis report to the user. Highlight:
- The overall score and verdict (Ready / Needs Refinement / Not Ready)
- Any CRITICAL or HIGH findings — these are the most important to address
- A quick summary of what's good and what needs work

## Step 4 — Suggest next steps

After presenting the results, guide the user on what to do next:

- If the story scored **≥ 7 and has no CRITICAL findings**:
  > "This story looks ready for the DoR gate. Run `/dor-gatekeeper [KEY]` to enforce the Definition of Ready and post the verdict to Jira."

- If the story scored **< 7 or has CRITICAL findings**:
  > "This story needs refinement before it can enter the sprint. Share the findings above with the product owner and re-analyze once the issues are addressed."

- If the user wants to run the full pipeline end-to-end:
  > "You can also run `/story-pipeline [KEY]` to execute the full pipeline — analysis, DoR gate, AC enrichment, risk scoring, and test case generation — all in one go."
