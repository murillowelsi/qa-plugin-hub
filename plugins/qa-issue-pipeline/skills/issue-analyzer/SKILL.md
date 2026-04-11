---
name: issue-analyzer
description: >
  Analyzes a Jira issue (story, bug, task, spike, subtask) using ISTQB-aligned quality checks — INVEST criteria,
  clarity, AC quality, testability, and completeness — then produces a structured
  report with severity-rated findings and an overall readiness score (0–10).
  Use this skill whenever the user provides a Jira ticket key and wants a quality
  review, asks "is this issue ready?", "is this story ready?", "can you check this ticket?", "review our
  AC", "is this testable?", or wants to prepare an issue for refinement or sprint
  planning. Trigger even if the user just pastes a ticket key with no extra context.
---

# Issue Analyzer

You are running the **issue-analyzer** skill. Evaluate a Jira issue for quality and readiness using ISTQB-aligned criteria. Do all the work directly — no sub-agents.

## Step 1 — Get the ticket key

If the user has not provided a Jira ticket key, ask for one now. Do not proceed without it.

## Step 2 — Fetch the ticket

Fetch the full story using the Jira MCP tool. If the story references linked Confluence pages, fetch those too for additional context.

## Step 3 — Analyze

Evaluate the story across every dimension below. For each one, assign a severity rating:

- **CRITICAL** — Blocks the story entirely, must be fixed before any work begins
- **HIGH** — Significant quality gap, likely to cause rework or missed coverage
- **MEDIUM** — Noticeable issue, should be addressed but won't block the sprint
- **LOW** — Minor improvement opportunity
- **OK** — Dimension is well-covered

### Dimensions

**1. INVEST Criteria**
- Independent: can this be developed and tested without depending on another in-progress story?
- Negotiable: is scope defined flexibly enough for the team to choose implementation?
- Valuable: is the business value clear? Who benefits and how?
- Estimable: does the team have enough information to size this story?
- Small: can this be completed within a single sprint?
- Testable: can you derive concrete acceptance criteria and test cases from it?

**2. Title Clarity**
- Must be understandable to any team member — technical or not
- Stories, Epics, and Bugs must NOT have technical scope tags as a prefix (`[BE]`, `[FE]`, `[API]`, `[iOS]`, `[Android]`) — flag as HIGH if found
- Tasks and Sub-tasks: tag prefixes are acceptable

**3. Clarity of Description**
Is the story written in clear, unambiguous language? Is the user role, goal, and rationale explicit ("As a [user], I want [goal] so that [reason]")?

**4. Acceptance Criteria Quality**
Are the ACs specific, measurable, and unambiguous? Do they cover the main success path? Are there obvious gaps (missing error handling, edge cases)?

**5. Testability**
Can you derive test cases directly from the ACs without guessing? Are there implicit assumptions that would block testing?

**6. Completeness**
Missing pieces: no ACs, no description, no story points, no linked designs or specs?

## Step 4 — Score

Compute an overall readiness score from 0 to 10:
- Start at 10
- Deduct 3 per CRITICAL finding
- Deduct 2 per HIGH finding
- Deduct 1 per MEDIUM finding
- Deduct 0.5 per LOW finding
- Minimum score is 0

**Verdict**:
- ≥ 8: Ready
- 5–7: Needs Refinement
- < 5: Not Ready

## Step 5 — Save the report

Save to `qa-output/issue-pipeline/<KEY>/01-analysis.md` using this structure:

```markdown
# Story Analysis — [TICKET-KEY]: [Story Title]

## Summary
- **Score**: X/10
- **Verdict**: Ready / Needs Refinement / Not Ready
- **Analyzed on**: [date]

## Findings

| Dimension | Rating | Finding |
|---|---|---|
| INVEST — Testable | CRITICAL | No acceptance criteria defined |
| Title Clarity | HIGH | Technical scope tag [BE] prefix found |
| AC Quality | OK | — |
| Testability | MEDIUM | AC #2 uses vague language ("should work correctly") |
| Completeness | LOW | No story points estimated |

## Details

### [Dimension Name]
[Expanded explanation of the finding and why it matters]

## Recommended Fixes
1. [Most severe issue — actionable fix]
2. [Next fix...]
```

## Step 6 — Present the results

Display the score, verdict, and findings table. Highlight CRITICAL and HIGH findings prominently.

Then guide the user:

- If score **≥ 7 and no CRITICAL findings**:
  > "This story looks ready for the DoR gate. Run `/dor-gatekeeper [KEY]` to enforce the Definition of Ready and post the verdict to Jira."

- If score **< 7 or has CRITICAL findings**:
  > "This story needs refinement before it can enter the sprint. Run `/issue-refiner [KEY]` to automatically rewrite it and fix the findings."

- If the user wants the full pipeline:
  > "You can also run `/issue-pipeline [KEY]` to execute the full pipeline end-to-end."
