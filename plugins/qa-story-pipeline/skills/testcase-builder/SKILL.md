---
name: testcase-builder
description: >
  Generates structured test cases from enriched ACs and risk scores. Covers
  positive, negative, boundary, and edge scenarios ordered by risk priority so
  the highest-risk areas are always tested first.
  Use this skill after risk-scorer, when the user wants test cases from a story,
  asks "generate test cases", "write test cases for this story", "create a test
  suite", "what should we test?", or wants to turn acceptance criteria into
  executable test cases. Also trigger when the user wants to know how many tests
  to write or what to cover for a given ticket.
---

# Test Case Builder

You are running the **testcase-builder** skill. Your job is to generate a complete, prioritized set of test cases from enriched ACs and risk scores.

## Step 1 — Get the ticket key

If the user has not provided a Jira ticket key, ask for one now.

## Step 2 — Check prerequisites

Look for both:
- `qa-output/story-pipeline/<KEY>/02-enriched-ac.md`
- `qa-output/story-pipeline/<KEY>/03-risk-matrix.md`

If either is missing, tell the user which step to run first:
- No enriched ACs → run `/ac-enricher [KEY]`
- No risk matrix → run `/risk-scorer [KEY]`

If only the enriched ACs exist (no risk matrix), offer to proceed without risk prioritization — but recommend running risk-scorer first for better results.

## Step 3 — Spawn the testcase-builder agent

Delegate to the **testcase-builder** agent, passing:
- The ticket key
- Path to enriched ACs: `qa-output/story-pipeline/<KEY>/02-enriched-ac.md`
- Path to risk matrix: `qa-output/story-pipeline/<KEY>/03-risk-matrix.md`

The agent will:
- Generate structured test cases ordered by risk priority (HIGH first)
- Cover all HIGH risk areas with at least 2 test cases each
- Use descriptive titles (no TC-001 IDs, no Gherkin)
- Include Preconditions, Steps, Expected Result, and Type for each case
- Save to `qa-output/story-pipeline/<KEY>/04-test-cases.md`

## Step 4 — Present the results

Show a summary:
- Total test cases generated
- Breakdown by type (Positive / Negative / Boundary / Edge)
- Breakdown by risk level (HIGH / MEDIUM / LOW)
- Preview of the first 2 test cases

## Step 5 — Suggest next steps

> "Test cases are ready at `qa-output/story-pipeline/[KEY]/04-test-cases.md`."
>
> "You can now:
> - Run `/story-pipeline [KEY]` to generate the full pipeline report and post it to Jira
> - Or use the test cases directly in your test management tool"
