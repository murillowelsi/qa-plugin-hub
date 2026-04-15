---
name: sprint-quality-gate
description: >
  Checks the quality of every ticket in a Jira sprint by spawning parallel agents —
  one per ticket — then aggregating all results into a sprint readiness report.
  Stories and Epics get the full 6-step pipeline; Bugs, Tasks, Spikes, and Sub-tasks
  get a focused quality analysis. Each ticket receives an individual Jira comment.
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

Display a confirmation table before proceeding:

| Key | Type | Summary | Status |
|-----|------|---------|--------|

Then say: "Found X tickets in Sprint N. Starting parallel quality gate — running all tickets simultaneously..."

---

## Step 3 — Spawn parallel agents

In a **single message**, use the Agent tool to spawn one subagent per ticket. All agents start at the same time — never wait for one to finish before spawning the others.

**Routing by issue type:**
- Story or Epic → `subagent_type: "ticket-pipeline-worker"`
- Bug, Task, Spike, or Sub-task → `subagent_type: "ticket-quality-checker"`

**Prompt template for each agent:**
```
Ticket key: <KEY>
Issue type: <TYPE>
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
