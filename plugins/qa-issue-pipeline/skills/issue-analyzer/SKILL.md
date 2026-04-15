---
name: issue-analyzer
description: >
  Analyzes a Jira issue (User Story, Bug, or Task) using ISTQB-aligned quality checks — INVEST criteria,
  clarity, AC quality, testability, and completeness — then produces a structured
  report with severity-rated findings and an overall readiness score (0–10).
  Use this skill whenever the user provides a Jira ticket key and wants a quality
  review, asks "is this issue ready?", "is this story ready?", "can you check this ticket?", "review our
  AC", "is this testable?", or wants to prepare an issue for refinement or sprint
  planning. Trigger even if the user just pastes a ticket key with no extra context.
allowed-tools: Read, Write, mcp__*__getJiraIssue, mcp__*__getConfluencePage, mcp__*__getJiraIssueRemoteIssueLinks, mcp__*__searchJiraIssuesUsingJql
---

# Issue Analyzer

You are running the **issue-analyzer** skill. Evaluate a Jira issue for quality and readiness using ISTQB-aligned criteria. Do all the work directly — no sub-agents.

## Step 1 — Get the ticket key

If the user has not provided a Jira ticket key, ask for one now. Do not proceed without it.

## Step 2 — Fetch the ticket

Fetch the full story using the Jira MCP tool. If the story references linked Confluence pages, fetch those too for additional context.

Also fetch the following when available — they are needed for dimension #8:
- **Parent ticket**: if this ticket has a parent, fetch it to understand the hierarchy and assess whether this ticket delivers value independently or belongs as a subtask.
- **Subtasks**: if this ticket has subtasks, fetch them to verify they represent implementation steps, not independently valuable deliverables.
- **Siblings**: if a parent exists, fetch its other children to detect technical-layer fragmentation (e.g., multiple sibling tasks for FE/BE/API/tests that should be subtasks of a shared story).

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
- No issue type may have a prefix tag in the title (`[BE]`, `[FE]`, `[API-CLIENT]`, `[E2E]`, `[Bug]`, `[iOS]`, `[Android]`, or similar) — flag as HIGH if found
- Technical area classification belongs in **Jira Components** and **Labels** — use only existing values from the ticket, do not create new ones

**3. Clarity of Description**
Is the story written in clear, unambiguous language? Is the user role, goal, and rationale explicit ("As a [user], I want [goal] so that [reason]")?

**4. Acceptance Criteria Quality**
Are the ACs specific, measurable, and unambiguous? Do they cover the main success path? Are there obvious gaps (missing error handling, edge cases)?

**5. Testability**
Can you derive test cases directly from the ACs without guessing? Are there implicit assumptions that would block testing?

**6. Completeness**
Missing pieces: no ACs, no description, no story points, no linked designs or specs?

**7. Issue Type Correctness**
Only three issue types are permitted: **User Story**, **Bug**, and **Task**. Flag as HIGH if any other type is used.

- **User Story** — represents new or modified functionality that directly impacts the user
- **Bug** — use **only** when a defect is identified in **production** and requires resolution. If the defect was found in an earlier stage (development, testing, or staging), it must be classified as a Task, not a Bug. Flag as HIGH if a Bug ticket describes a pre-production finding.
- **Task** — use for issues found in pre-production environments (development, testing, staging), and for internal or supporting activities that are not on the roadmap and not directly user-facing

**8. Work Item Structure & Estimation**

The goal here is to detect tickets that misrepresent how work is decomposed — either by creating standalone tasks for things that should be subtasks, or by fragmenting a single deliverable into multiple tickets for the wrong reasons.

- **Deliverable independence**: Does this ticket represent a clear, independently testable and potentially releasable outcome? If not — for example, it's a purely technical layer like "implement API endpoint" or "write frontend component" with no independent user value — it likely belongs as a subtask of a parent story. Flag as HIGH if found standalone.
- **Estimation level**: Only parent tickets (User Stories, Tasks) should carry story points. Subtasks must not be estimated. Flag as HIGH if this ticket is a subtask with story points assigned. (Missing story points on a parent is already covered by dimension #6 Completeness.)
- **Technical-layer fragmentation**: If sibling tickets exist that together cover the same deliverable split by technical layer (FE, BE, API, tests, etc.) rather than by user value, this is an anti-pattern. Each layer should be a subtask, not a standalone ticket. Flag as HIGH if detected.
- **Burndown anti-pattern**: Flag as MEDIUM if the ticket appears created primarily to log activity rather than deliver value — e.g., a title like "Track progress for X", "Create subtasks for Y", or a description with no concrete outcome.
- **Subtask scope correctness**: If this ticket has subtasks, each subtask should represent an implementation step (API work, UI, tests, refactoring). If a subtask looks like it could be independently released and validated, it should be promoted to its own ticket. Flag as MEDIUM if found.

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

- If there are **HIGH findings on dimension #8 (Work Item Structure)** — specifically deliverable independence, technical-layer fragmentation, or subtask misclassification:
  > "This ticket has structural issues. Run `/ticket-splitter [KEY]` to get a concrete proposal for how to split or reorganise it into properly scoped stories."

- If the user wants the full pipeline:
  > "You can also run `/issue-pipeline [KEY]` to execute the full pipeline end-to-end."
