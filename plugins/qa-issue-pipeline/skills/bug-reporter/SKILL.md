---
name: bug-reporter
description: >
  Creates a complete, structured bug report from a bug description or Jira ticket,
  then posts it as a Jira comment or creates a new Jira bug issue.
  Use this skill whenever the user says "write a bug report", "create a bug report for",
  "log this bug", "I found a bug", "document this defect", "file a bug",
  "report a bug on [KEY]", or describes a defect and wants it documented or tracked.
  Also trigger when the user pastes raw bug observations and wants them turned into a
  proper report, or when they ask "can you write this up as a bug?" — even without
  a Jira key. This is the go-to skill for any defect documentation workflow.
---

# Bug Reporter

You are running the **bug-reporter** skill. Produce a complete, structured bug report and post it to Jira (as a comment on an existing ticket, or as a new bug issue). Do all the work directly — no sub-agents.

## Step 1 — Collect the bug information

Two input modes:

**Mode A — Jira ticket key provided**
Fetch the ticket using the Jira MCP tool. Extract whatever bug information is already present: summary, description, steps, environment, expected/actual results.

**Mode B — Free-text description**
The user has described the bug in plain language. Extract what they've given you. Don't ask for things they've clearly already provided.

After extracting what you have, check for mandatory fields. If either is missing, ask for it before proceeding:
- **Steps to reproduce** — without these the bug can't be verified
- **Actual result** — what the system actually does wrong

For optional fields (environment, root cause, priority), ask in a single prompt and accept "unknown" or a skip as a valid answer. Don't block progress on optional fields.

## Step 2 — Build the structured report

Construct the full bug report using this template:

```markdown
# Bug Report — [KEY or slug]: [Summary]

**Reported on**: [date]
**Reported by**: [user name or "QA Team"]
**Status**: Open

## Bug Details

| Field       | Value                          |
|-------------|--------------------------------|
| Severity    | CRITICAL / HIGH / MEDIUM / LOW |
| Priority    | P1 / P2 / P3 / P4              |
| Environment | [browser, OS, app version]     |
| Component   | [affected feature or area]     |

## Preconditions
- [State the app/system must be in before reproducing]

## Steps to Reproduce
1. [First action]
2. [Second action]
3. [Continue...]

## Expected Result
[What the system should do]

## Actual Result
[What the system actually does — be specific, no "it doesn't work"]

## Root Cause Hypothesis
[Tester's best guess at what's broken, or "Unknown"]

## Attachments
[ ] Screenshot of the issue
[ ] Browser console logs
[ ] Network requests (HAR file)
[ ] Test data used
```

**Writing good summaries**: The summary should be one line, specific, and self-explanatory. Bad: "Login doesn't work". Good: "Login fails with 500 error when password contains special characters on Chrome 123".

## Step 3 — Assess severity and priority

Suggest severity and priority based on the bug description, using these ISTQB-aligned definitions:

**Severity**
| Level    | Definition |
|----------|------------|
| CRITICAL | System crash, data loss, security vulnerability, or complete feature failure blocking a core user journey |
| HIGH     | Major feature broken, no workaround available, significant user impact |
| MEDIUM   | Feature partially broken, a workaround exists |
| LOW      | Cosmetic issue, minor UX problem, typo |

**Priority** maps to urgency for the team:
- **P1** — Fix immediately (production blocker or CRITICAL severity)
- **P2** — Fix in current sprint (HIGH severity or customer-facing regression)
- **P3** — Fix in next sprint (MEDIUM severity or low user impact)
- **P4** — Fix when convenient (LOW severity, cosmetic)

Show your severity/priority recommendation with a one-sentence rationale, then ask the user to confirm or override before posting to Jira.

## Step 4 — Save locally

Always save a local copy before posting to Jira:
`qa-output/bug-reports/<KEY-or-slug>/bug-report.md`

Use the ticket key if available. If not, derive a short slug from the summary (e.g., `login-500-special-chars`).

## Step 5 — Post to Jira

Ask the user which action they want:

**Option A — Post as comment on existing ticket**
Post the following formatted comment using the Jira MCP tool:

```
🐛 *Bug Report*

*Summary*: [one-liner]
*Severity*: [level] | *Priority*: [P1–P4]

*Environment*: [browser / OS / version]
*Component*: [affected area]

*Preconditions*:
[list]

*Steps to Reproduce*:
1. [step]
2. [step]

*Expected*: [result]
*Actual*: [result]

*Root Cause Hypothesis*: [guess or Unknown]

_Reported by QA Bug Reporter on [date]_
```

**Option B — Create a new Jira bug issue**
If no ticket exists, ask for the Jira project key and create a new issue using the Jira MCP tool with:
- Issue type: Bug
- Summary: the one-line bug summary
- Description: the full structured report in Jira markup
- Priority: as determined in Step 3

## Step 6 — Present next steps

After posting, suggest follow-up actions based on context:

- If the bug reveals missing or ambiguous acceptance criteria on the related story:
  > "This bug may point to a story quality problem. Run `/issue-analyzer [KEY]` to review the ACs and testability of the parent story."

- If the bug area needs regression coverage:
  > "Want to make sure this doesn't regress? Run `/testcase-builder [KEY]` to generate test cases for this functional area."

- If this is part of a broader story pipeline:
  > "You can also run `/issue-pipeline [KEY]` to execute the full QA pipeline on the related story."
