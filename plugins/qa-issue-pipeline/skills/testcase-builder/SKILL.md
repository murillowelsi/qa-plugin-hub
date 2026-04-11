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

You are running the **testcase-builder** skill. Generate a complete, prioritized set of test cases from enriched ACs and risk scores. Do all the work directly — no sub-agents.

## Step 1 — Get the ticket key

If the user has not provided a Jira ticket key, ask for one now.

## Step 2 — Check prerequisites

Look for both:
- `qa-output/issue-pipeline/<KEY>/02-enriched-ac.md`
- `qa-output/issue-pipeline/<KEY>/03-risk-matrix.md`

If either is missing:
- No enriched ACs → tell the user to run `/ac-enricher [KEY]` first
- No risk matrix → offer to proceed without risk prioritization, but recommend `/risk-scorer [KEY]` first for better results

Read both files now.

## Step 3 — Generate test cases

Use the risk matrix to determine order — HIGH risk areas first, then MEDIUM, then LOW.

**Rules**:
- No Gherkin (no Given/When/Then — that's for BDD scenarios, not test cases)
- No TC-001 IDs — use descriptive titles that read like a sentence
- Each title must stand alone: a reader should understand what is being verified from the title alone
- Cover all HIGH risk areas with at least 2 test cases each
- Cover MEDIUM areas with at least 1 test case each
- LOW areas: 1 test case or note as lower priority

**Test case format**:
```
### [Descriptive title of what is being verified]
**Type**: Positive | Negative | Boundary | Edge
**Risk Area**: [Functional area from risk matrix]
**Risk Level**: HIGH | MEDIUM | LOW

**Preconditions**:
- [What must be true before the test starts]

**Steps**:
1. [First action]
2. [Second action]

**Expected Result**:
[What the system should do or display when steps are completed correctly]
```

## Step 4 — Save output

Save to `qa-output/issue-pipeline/<KEY>/04-test-cases.md`:

```markdown
# Test Cases — [TICKET-KEY]: [Story Title]

## Summary
- Total test cases: X
- Positive: X | Negative: X | Boundary: X | Edge: X
- HIGH risk coverage: X cases across Y areas
- MEDIUM risk coverage: X cases across Y areas
- LOW risk coverage: X cases across Y areas

## Test Cases by Risk Area

### [HIGH] [Area Name]
[test cases...]

### [MEDIUM] [Area Name]
[test cases...]
```

## Step 5 — Present the results

Show:
- Total test cases generated
- Breakdown by type (Positive / Negative / Boundary / Edge)
- Breakdown by risk level
- Preview of the first 2 test cases

Then suggest next steps:
> "Test cases are ready at `qa-output/issue-pipeline/[KEY]/04-test-cases.md`."
> "Next: run `/automation-planner [KEY]` to triage which test cases should become Playwright specs, API tests, or stay manual — before writing any automation."
