---
name: dor-gatekeeper
description: >
  Enforces the Definition of Ready (DoR) on a Jira user story. Reads the story
  analysis report, applies DoR rules (score threshold, no CRITICAL findings,
  minimum ACs, estimability), posts a structured Jira comment with the verdict
  and actionable fixes, and returns PASS or BLOCK.
  Use this skill after issue-analyzer, when the user wants to gate a story before
  sprint entry, asks "is this story ready for the sprint?", "can we pull this in?",
  "does this meet our DoR?", "check the definition of ready", or wants to post a QA verdict to Jira.
  Always use this skill — not manual judgement — whenever a DoR check is needed.
allowed-tools: Read, Write, mcp__*__getJiraIssue, mcp__*__addCommentToJiraIssue
---

# DoR Gatekeeper

You are running the **dor-gatekeeper** skill. Enforce the team's Definition of Ready on a Jira user story and communicate the verdict to Jira. Do all the work directly — no sub-agents.

## Step 1 — Get the ticket key

If the user has not provided a Jira ticket key, ask for one now.

## Step 2 — Check for analysis report

Look for `qa-output/issue-pipeline/<KEY>/01-analysis.md`.

If it does not exist:
> "I need the story analysis before I can check the DoR. Run `/issue-analyzer [KEY]` first, then come back here."

Do not proceed without the analysis report. Read it now.

## Step 3 — Apply DoR rules

Apply the following rules in order. The story must pass ALL of them for a PASS verdict:

| Rule | Requirement |
|---|---|
| Score threshold | Overall readiness score must be ≥ 7/10 |
| No CRITICAL findings | Zero CRITICAL severity findings allowed |
| Minimum ACs | At least 2 acceptance criteria must be present and clear |
| Estimable | Story must have enough information to be estimated |
| Clear description | Story description must be present and understandable |
| Work item structure | Ticket must represent independently deliverable value. No HIGH findings on technical-layer fragmentation, subtask misclassification, or estimation at the wrong level. |

## Step 4 — Preview and confirm before posting to Jira

Format the verdict comment as shown below, then display it to the user in full and ask for approval before posting anything.

**If PASS**:
```
✅ *Story approved for sprint — DoR met.*

Readiness score: X/10
All Definition of Ready criteria passed.

_Reviewed by QA Issue Pipeline on [date]_
```

**If BLOCK**:
```
🚫 *Story blocked from sprint — DoR not met.*

Readiness score: X/10

*Failing rules:*
- ❌ [Rule name]: [Why it failed and what needs to be fixed]

*Required actions before this story can enter sprint:*
1. [Specific fix #1]
2. [Specific fix #2]

_Reviewed by QA Issue Pipeline on [date]_
```

Be specific and actionable. The person reading the comment should know exactly what to fix without asking follow-up questions.

After formatting the comment, show it to the user and ask:

> "Here is the comment that will be posted to [KEY]. Approve to post, tell me what to change, or reply **skip** to not post to Jira."

**Do not post to Jira until the user explicitly approves.** If the user requests changes, revise the comment and ask again.

## Step 5 — Post to Jira (only after approval)

Once the user approves, post the comment using the Jira MCP tool.

## Step 6 — Save verdict

Save to `qa-output/issue-pipeline/<KEY>/dor-verdict.json`:

```json
{
  "ticket": "PROJ-123",
  "verdict": "PASS",
  "score": 8.5,
  "failing_rules": [],
  "evaluated_on": "2026-04-10"
}
```

For BLOCK, populate `failing_rules` with the names of the rules that failed.

## Step 7 — Present the verdict

**If PASS**:
> "✅ Story [KEY] passed the DoR gate (score: X/10). The Jira ticket has been updated with the approval comment."
> "Next step: run `/ac-enricher [KEY]` to rewrite the ACs into structured BDD scenarios."

**If BLOCK**:
> "🚫 Story [KEY] is blocked from the sprint. Failing rules: [list]. The Jira ticket has been updated."
> "Run `/issue-refiner [KEY]` to automatically rewrite the story and fix all findings, then re-run `/dor-gatekeeper [KEY]`."
