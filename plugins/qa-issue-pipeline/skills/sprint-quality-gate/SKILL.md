---
name: sprint-quality-gate
description: >
  Checks the quality of every ticket in a Jira sprint by spawning parallel agents —
  one per ticket — then aggregating all results into a sprint readiness report.
  Only User Stories, Bugs, and Tasks are supported. Epics, Spikes, Sub-tasks, and
  other types are flagged as type violations with a reclassification recommendation
  posted as a Jira comment — they do not receive a quality score.
  Stories get the full 6-step pipeline; Bugs and Tasks get a focused quality analysis.
  Each ticket receives an individual Jira comment.
  Use this skill whenever the user provides a sprint number and wants to check sprint
  quality, assess sprint readiness before it starts, run the pipeline across an entire
  sprint, or see which tickets are ready versus blocked.
  Trigger on: "check sprint N", "run quality gate on sprint", "analyze sprint N",
  "is sprint N ready", "run the pipeline on all sprint tickets", "sprint quality check",
  "sprint readiness report".
allowed-tools: Read, Write, mcp__*__searchJiraIssuesUsingJql, mcp__*__getJiraIssue, mcp__*__addCommentToJiraIssue
---

# Sprint Quality Gate

Run the full QA pipeline across every ticket in a sprint simultaneously, then consolidate results into a sprint readiness report.

## Step 1 — Get the sprint number

If the sprint number was not provided, ask for it now. Do not proceed without it.

---

## Step 2 — Fetch sprint tickets

Search Jira using:
```
sprint = <sprint_number> AND issueType in (Story, Epic, Bug, Task, Spike, "Sub-task")
```

Retrieve for each ticket: `key`, `summary`, `issuetype`, `status`, `assignee`, `priority`.

Partition results into two groups before proceeding:
- **Supported**: Story, Bug, Task — these will be processed by agents
- **Unsupported**: Spike, Sub-task, Epic, and any other types — these will receive a type violation flag instead of a quality analysis

Display a confirmation table for all tickets (both groups):

| Key | Type | Summary | Status |
|-----|------|---------|--------|

Then say: "Found X tickets in Sprint N (Y supported, Z flagged as unsupported type). Starting parallel quality gate..."

---

## Step 3 — Spawn parallel agents

In a **single message**, use the Agent tool to spawn one subagent per ticket. All agents start at the same time — never wait for one to finish before spawning the others.

**Routing by issue type:**
- Story → `subagent_type: "ticket-pipeline-worker"`
- Bug or Task → `subagent_type: "ticket-quality-checker"`
- Spike, Sub-task, Epic, or any other unsupported type → `subagent_type: "ticket-quality-checker"`

**Prompt template for supported types (Story, Bug, Task):**
```
Ticket key: <KEY>
Issue type: <TYPE>
```

**Prompt template for unsupported types:**
```
Ticket key: <KEY>
Issue type: <TYPE>

This ticket has an unsupported issue type. Fetch the ticket, read its title and description, then:
1. Flag the type violation clearly
2. Recommend the correct type (User Story, Bug, or Task) based on the content, with a 1–2 sentence rationale
3. Post a Jira comment with the violation and recommendation
4. Return the JSON summary with score: 0, status: "BLOCK", top_issues: ["Issue type '[TYPE]' is not permitted — recommend reclassifying as [suggested type]"]
```

---

## Step 4 — Collect results

Wait for all agents to complete. Each agent returns a JSON summary as its final line:
```json
{"key":"KEY","type":"TYPE","score":N,"status":"PASS|BLOCK","top_issues":["..."]}
```

Parse the JSON from each agent's response to extract the summary data.

---

## Step 5 — Aggregate

From the collected summaries, compute:
- Total tickets processed
- PASS count and percentage
- BLOCK count and percentage
- Average readiness score (across all tickets)
- Worst ticket: lowest score and its key
- **Common failure patterns**: group all `top_issues` entries across BLOCK tickets by category, ranked by frequency

---

## Step 6 — Sprint report

Save to `qa-output/sprint-quality/sprint-<N>/sprint-report.md`:

```markdown
# Sprint <N> Quality Gate Report

**Date**: [date] | **Total tickets**: X | **Pass rate**: X%

---

## Summary

| Metric | Value |
|--------|-------|
| ✅ PASS | X (X%) |
| 🚫 BLOCK | X (X%) |
| Avg score | X.X / 10 |
| Worst ticket | KEY — X/10 |

---

## Ticket Breakdown

| Key | Type | Score | Status | Top Issues |
|-----|------|-------|--------|------------|
| KEY | Story | X.X | ✅ PASS | — |
| KEY | Bug | X.X | 🚫 BLOCK | [finding 1], [finding 2] |

---

## Common Failure Patterns

[Group top_issues by category across all blocked tickets, ranked by frequency]
[e.g. "Missing acceptance criteria — 4 tickets", "No story points — 3 tickets"]

---

## Recommended Actions

[Prioritised action list — which tickets need what, most critical first]
[Focus on blockers that affect the most tickets or highest-risk stories]
```

Display the full report in the conversation after saving it.
