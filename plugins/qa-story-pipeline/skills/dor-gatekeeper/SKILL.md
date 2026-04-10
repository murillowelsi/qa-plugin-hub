---
name: dor-gatekeeper
description: >
  Enforces the Definition of Ready (DoR) on a Jira user story. Reads the story
  analysis report, applies DoR rules (score threshold, no CRITICAL findings,
  minimum ACs, estimability), posts a structured Jira comment with the verdict
  and actionable fixes, and returns PASS or BLOCK.
  Use this skill after story-analyzer, when the user wants to gate a story before
  sprint entry, asks "is this story ready for the sprint?", "can we pull this in?",
  "does this meet our DoR?", or wants to post a QA verdict to Jira.
---

# DoR Gatekeeper

You are running the **dor-gatekeeper** skill. Your job is to enforce the team's Definition of Ready on a Jira user story and communicate the verdict clearly.

## Step 1 — Get the ticket key

If the user has not provided a Jira ticket key, ask for one now.

## Step 2 — Check for analysis report

Look for `qa-output/story-pipeline/<KEY>/01-analysis.md`.

If it does not exist, the story has not been analyzed yet. Tell the user:
> "I need to analyze this story before I can check the DoR. Run `/story-analyzer [KEY]` first, then come back here."

Do not proceed without the analysis report.

## Step 3 — Spawn the dor-gatekeeper agent

Delegate to the **dor-gatekeeper** agent, passing:
- The ticket key
- The path to the analysis report

The agent will:
- Apply DoR rules (score ≥ 7, no CRITICAL findings, at least 2 ACs, estimable)
- Post a structured comment to the Jira ticket (PASS or BLOCK with reasons)
- Save the verdict to `qa-output/story-pipeline/<KEY>/dor-verdict.json`

## Step 4 — Present the verdict

Display the verdict clearly:

**If PASS**:
> "✅ Story [KEY] passed the DoR gate (score: X/10). The Jira ticket has been updated with the approval comment."
> "Next step: run `/ac-enricher [KEY]` to rewrite the ACs into structured BDD scenarios."

**If BLOCK**:
> "🚫 Story [KEY] is blocked from the sprint. The following DoR rules failed: [list them]. The Jira ticket has been updated with the details and recommended fixes."
> "Once the product owner addresses the issues, re-run `/story-analyzer [KEY]` and then `/dor-gatekeeper [KEY]` again."
