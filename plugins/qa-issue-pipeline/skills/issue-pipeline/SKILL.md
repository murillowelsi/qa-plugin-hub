---
name: issue-pipeline
description: >
  Runs the full automated issue quality pipeline end-to-end without user
  hand-holding: story analysis → DoR gate → AC enrichment + risk scoring
  (parallel) → test case generation → pipeline report posted to Jira.
  Use this skill whenever the user says "run the pipeline", "full issue pipeline", "full story pipeline",
  "process this issue", "process this story", "do everything for this ticket", "run QA on this issue", "run QA on this story",
  or provides a ticket key and clearly wants the complete workflow executed
  automatically. Also trigger when the user asks to "analyze and generate test
  cases" or any phrase that implies running multiple pipeline steps together.
  This is the main entry point for a single ticket — prefer this over running
  individual skills one by one. For running the pipeline across an entire sprint,
  use the sprint-quality-gate skill instead.
allowed-tools: Read, Write, mcp__*__getJiraIssue, mcp__*__getConfluencePage, mcp__*__addCommentToJiraIssue, mcp__*__searchJiraIssuesUsingJql, mcp__*__getJiraIssueRemoteIssueLinks
---

# Issue Pipeline — Orchestrator

You are running the **issue-pipeline** skill. Execute the full pipeline for a single ticket. Steps 1 and 2 run inline; steps 3 and 4 run in parallel via agents; steps 5 and 6 run inline after the parallel steps complete. You have full MCP access — use it directly at every step that requires Jira interaction.

## Step 0 — Get the ticket key

If the user has not provided a Jira ticket key, ask for one now. Do not proceed without it.

Report progress after each step using this format:
> `[X/6] [Step name] — running...`
> `[X/6] [Step name] — ✅ done ([key metric])`

---

## [1/6] Story Analysis

Fetch the full Jira ticket using the MCP tool. If the story references linked Confluence pages, fetch those too.

**Issue Type Guard**: Check the `issuetype` field. Only **User Story**, **Bug**, and **Task** are permitted.

If the type is anything else, do not run the pipeline. Instead, read the ticket's title and description to understand its nature, then produce a reclassification recommendation:

**Reclassification logic:**
- **Spike** → almost always should be a **Task** (investigation work is internal, non-user-facing, pre-production)
- **Sub-task** → flag the parent story for the pipeline instead; sub-tasks are not independently processed
- **Epic** → decompose into **User Stories** first, then run the pipeline on each story
- **Any other type** → recommend the closest match (Story, Bug, or Task) based on the content

Respond with:

> "**Issue Type Violation — [KEY]: [Title]**
>
> Current type: **[Type]** — this is not a permitted issue type.
> Only **User Story**, **Bug**, and **Task** are allowed in this pipeline.
>
> **Recommended reclassification: [Suggested type]**
> Reason: [1–2 sentences explaining why, based on the ticket's actual content]
>
> To proceed: change the issue type in Jira to [Suggested type], then re-run `/issue-pipeline [KEY]`."

Stop here — do not continue to analysis or any subsequent pipeline steps for unsupported types.

Evaluate the story across every dimension below. Assign a severity rating to each:
- **CRITICAL** — must be fixed before any work begins
- **HIGH** — significant quality gap, likely to cause rework
- **MEDIUM** — noticeable issue, should be addressed
- **LOW** — minor improvement opportunity
- **OK** — well-covered

**Dimensions:**

1. **INVEST Criteria** — Independent, Negotiable, Valuable, Estimable, Small, Testable
2. **Title Clarity** — understandable to any team member; no prefix tags (`[BE]`, `[FE]`, `[API-CLIENT]`, `[E2E]`, `[Bug]`) on any issue type (flag as HIGH if found); technical area classification belongs in Jira Components and Labels (existing values only — do not create new ones)
3. **Clarity of Description** — clear user role, goal, rationale; "As a / I want / So that" structure
4. **Acceptance Criteria Quality** — specific, measurable, unambiguous; covers main path; no obvious gaps
5. **Testability** — test cases derivable directly from ACs without guessing
6. **Completeness** — no missing ACs, description, story points, or linked designs
7. **Issue Type Correctness** — only **User Story**, **Bug**, and **Task** are permitted. Any other type is rejected by the Issue Type Guard at the top of this step before analysis begins. Within supported types, verify correct classification:
   - **User Story**: new or modified functionality that directly impacts the user
   - **Bug**: use **only** for defects found in **production**. If the defect was found in development, testing, or staging, it must be a Task — flag as HIGH if misclassified
   - **Task**: issues from pre-production environments, or internal/supporting activities not on the roadmap and not directly user-facing
8. **Work Item Structure & Estimation** — fetch the parent ticket, subtasks, and sibling tickets when available, then evaluate:
   - **Deliverable independence**: does this ticket represent a clear, independently testable and releasable outcome? Flag as HIGH if it's a pure technical layer (e.g., "implement API endpoint", "write frontend component") with no standalone user value
   - **Estimation level**: only parent tickets carry story points — subtasks must not be estimated. Flag as HIGH if this ticket is a subtask with story points assigned
   - **Technical-layer fragmentation**: flag as HIGH if sibling tickets split the same deliverable by technical layer (FE/BE/API/tests) rather than by user value — those should be subtasks, not separate tickets
   - **Burndown anti-pattern**: flag as MEDIUM if the ticket appears created primarily to log activity rather than deliver value (e.g., "Track progress for X", "Create subtasks for Y")
   - **Subtask scope correctness**: flag as MEDIUM if any subtask looks like it could be independently released and validated — it should be promoted to its own story

**Scoring:**
- Start at 10
- Deduct 3 per CRITICAL, 2 per HIGH, 1 per MEDIUM, 0.5 per LOW
- Minimum: 0

**Verdict:** ≥ 8 = Ready | 5–7 = Needs Refinement | < 5 = Not Ready

Save the report to `qa-output/issue-pipeline/<KEY>/01-analysis.md`:

```markdown
# Story Analysis — [KEY]: [Story Title]

## Summary
- **Score**: X/10
- **Verdict**: [verdict]
- **Analyzed on**: [date]

## Findings

| Dimension | Rating | Finding |
|---|---|---|
| ... | ... | ... |

## Details
[Expanded explanation per finding]

## Recommended Fixes
1. [Most severe — actionable fix]
```

Report: `[1/6] Story Analysis — ✅ done (score: X/10 — [verdict])`

---

## [2/6] DoR Gate

Apply these rules to the analysis report. The story must pass ALL of them:

| Rule | Requirement |
|---|---|
| Score threshold | ≥ 7/10 |
| No CRITICAL findings | Zero CRITICALs allowed |
| Minimum ACs | At least 2 clear acceptance criteria |
| Estimable | Enough information to size the story |
| Clear description | Description present and understandable |
| Work item structure | Ticket must represent independently deliverable value; no HIGH findings on technical-layer fragmentation, subtask misclassification, or estimation at the wrong level |

Save the verdict to `qa-output/issue-pipeline/<KEY>/dor-verdict.json`:
```json
{
  "ticket": "KEY",
  "verdict": "PASS or BLOCK",
  "score": 0.0,
  "failing_rules": [],
  "evaluated_on": "date"
}
```

Format the verdict comment as shown below and display it to the user for approval before posting anything.

**If PASS:**
```
✅ *Story approved for sprint — DoR met.*

Readiness score: X/10
All Definition of Ready criteria passed.

_Reviewed by QA Issue Pipeline on [date]_
```

**If BLOCK:**
```
🚫 *Story blocked from sprint — DoR not met.*

Readiness score: X/10

*Failing rules:*
- ❌ [Rule]: [Why it failed and what needs fixing]

*Required actions:*
1. [Specific fix]

_Reviewed by QA Issue Pipeline on [date]_
```

Show the formatted comment and ask: "Approve to post this to Jira, or reply **skip** to continue without posting."

**Do not post to Jira until the user explicitly approves.**

**If BLOCK — stop the pipeline here after the user responds.** Report:
> `[2/6] DoR Gate — 🚫 BLOCKED`
> "The story did not meet the Definition of Ready."
> "Run `/issue-refiner [KEY]` to automatically rewrite and fix all findings, then re-run `/issue-pipeline [KEY]`."

If the block includes a failing **Work item structure** rule:
> "This ticket has structural issues. Run `/ticket-splitter [KEY]` to get a concrete proposal for how to split or reorganise it before refining."

Show the failing rules and stop.

**If PASS:** Report: `[2/6] DoR Gate — ✅ PASS (Jira updated)`

---

## [3+4/6] AC Enrichment + Risk Scoring — parallel

These two steps are independent — both work from the ticket data and analysis already in context. Spawn them simultaneously in a **single message** using the Agent tool, then wait for both to finish before continuing.

Report before spawning: `[3+4/6] AC Enrichment + Risk Scoring — running in parallel...`

---

**Agent A — AC Enrichment**

Prompt:
```
Ticket key: <KEY>
The Jira ticket data and analysis report are already saved at qa-output/issue-pipeline/<KEY>/01-analysis.md.
Fetch the ticket from Jira to get the original acceptance criteria.

For each original AC:
1. Rewrite as Given/When/Then — standalone and self-explanatory
2. Add at least one negative path — wrong, missing, or invalid input
3. Add at least one edge case — boundary values, empty states, concurrent actions, limits

Scenario format:
### Scenario: [Short descriptive title]
**Type**: Positive | Negative | Edge Case
Given [precondition]
When [action]
Then [outcome]
And [additional outcome if needed]

Group scenarios under the original AC they derive from.
Save to qa-output/issue-pipeline/<KEY>/02-enriched-ac.md
```

---

**Agent B — Risk Scoring**

Prompt:
```
Ticket key: <KEY>
Read the analysis report at qa-output/issue-pipeline/<KEY>/01-analysis.md.
Fetch the ticket from Jira to get the original acceptance criteria and story description.

Group the story's functional areas into themes (e.g. "cancellation flow", "notification handling", "error states").

For each area score:
- Likelihood of failure (1–5): 1 = very unlikely → 5 = very likely
- Business impact (1–5): 1 = minimal → 5 = data loss / security / blocks core journey
- Risk score = Likelihood × Impact
- HIGH ≥ 15 | MEDIUM 8–14 | LOW ≤ 7

Save to qa-output/issue-pipeline/<KEY>/03-risk-matrix.md using this structure:
# Risk Matrix — [KEY]: [Story Title]
## Risk Summary
- HIGH: X | MEDIUM: X | LOW: X
## Risk Matrix
| # | Functional Area | Likelihood | Impact | Score | Level |
|---|---|---|---|---|---|
## Recommended Test Execution Order
1. **[Area]** — HIGH (Score: XX) — [Why critical]
## Risk Rationale
### [Area] — HIGH
[2–3 sentences on likelihood and business impact]
```

---

Wait for both agents to complete, then read both output files before continuing.

Report: `[3/6] AC Enrichment — ✅ done (X scenarios: X positive, X negative, X edge)`
Report: `[4/6] Risk Scoring — ✅ done (X HIGH, X MEDIUM, X LOW)`

---

## [5/6] Test Case Generation

Read the enriched ACs and risk matrix. Generate test cases ordered by risk priority — HIGH first, then MEDIUM, then LOW.

Rules:
- No Gherkin syntax — test cases, not BDD scenarios
- No TC-001 IDs — use descriptive titles that stand alone
- HIGH risk areas: at least 2 test cases each
- MEDIUM areas: at least 1 test case each
- LOW areas: 1 test case or note as lower priority

Format:
```
### [Descriptive title of what is being verified]
**Type**: Positive | Negative | Boundary | Edge
**Risk Area**: [area]
**Risk Level**: HIGH | MEDIUM | LOW

**Preconditions**:
- [required state or test data]

**Steps**:
1. [action]
2. [action]

**Expected Result**:
[what the system should do]
```

Save to `qa-output/issue-pipeline/<KEY>/04-test-cases.md` with a summary table at the top.

Report: `[5/6] Test Cases — ✅ done (X cases generated)`

---

## [6/6] Pipeline Report

Read all output files from `qa-output/issue-pipeline/<KEY>/`. Extract key metrics from each.

Save to `qa-output/issue-pipeline/<KEY>/05-pipeline-report.md`:

```markdown
# Pipeline Report — [KEY]: [Story Title]
**Run date**: [date]
**Pipeline status**: ✅ Complete

## Results at a Glance

| Step | Status | Key Metric |
|---|---|---|
| Story Analysis | ✅ Done | Score: X/10 — [verdict] |
| DoR Gate | ✅ PASS | All rules met |
| AC Enrichment | ✅ Done | X scenarios (X pos, X neg, X edge) |
| Risk Scoring | ✅ Done | X HIGH, X MEDIUM, X LOW |
| Test Cases | ✅ Done | X cases generated |

## Top Risk Areas
[HIGH risk areas from the risk matrix with scores]

## Recommended Next Steps
[2–3 concrete next actions for the team]
```

Format the following Jira comment and display it to the user for approval before posting:

```
📋 *QA Issue Pipeline — Complete*

*Story Analysis*: X/10 — [verdict]
*DoR Gate*: ✅ PASS
*AC Scenarios*: X generated
*Risk Areas*: X HIGH | X MEDIUM | X LOW
*Test Cases*: X generated

*Top risks*:
• [Area 1] — HIGH (Score: XX)
• [Area 2] — HIGH (Score: XX)

All outputs saved to qa-output/issue-pipeline/[KEY]/

_Run by QA Issue Pipeline on [date]_
```

Ask: "Approve to post this summary to Jira, or reply **skip** to finish without posting."

**Do not post to Jira until the user explicitly approves.** Once approved, post using the Jira MCP tool.

Report: `[6/6] Pipeline Report — ✅ done (Jira updated)`

---

## Final Summary

Display the full pipeline summary table:

```
╔══════════════════════════════════════════════════════════╗
║         QA Issue Pipeline — [KEY]: [Story Title]         ║
╠══════════════╦═══════════════╦══════════════════════════╣
║ Step         ║ Status        ║ Result                   ║
╠══════════════╬═══════════════╬══════════════════════════╣
║ Analysis     ║ ✅ Done       ║ Score: X/10 — [verdict]  ║
║ DoR Gate     ║ ✅ PASS       ║ Jira notified            ║
║ AC Enrichment║ ✅ Done       ║ X scenarios              ║
║ Risk Scoring ║ ✅ Done       ║ X HIGH / X MED / X LOW   ║
║ Test Cases   ║ ✅ Done       ║ X cases                  ║
║ Report       ║ ✅ Done       ║ Jira notified            ║
╚══════════════╩═══════════════╩══════════════════════════╝

All outputs saved to: qa-output/issue-pipeline/[KEY]/
```

## Guiding principles

- Never wait for the user between steps — run the full pipeline autonomously
- Use MCP tools directly at every step that needs Jira — no pre-fetching workarounds needed
- The DoR BLOCK is the only legitimate stopping point mid-pipeline
- If a step fails unexpectedly, report the error and ask the user how to proceed
