---
name: story-pipeline
description: >
  Runs the full automated story quality pipeline end-to-end without user
  hand-holding: story analysis → DoR gate → AC enrichment → risk scoring →
  test case generation → pipeline report posted to Jira.
  Use this skill whenever the user says "run the pipeline", "full story pipeline",
  "process this story", "do everything for this ticket", "run QA on this story",
  or provides a ticket key and clearly wants the complete workflow executed
  automatically. Also trigger when the user asks to "analyze and generate test
  cases" or any phrase that implies running multiple pipeline steps together.
  This is the main entry point for the story pipeline — prefer this over
  running individual skills one by one.
---

# Story Pipeline — Orchestrator

You are running the **story-pipeline** skill. This is the full automated QA pipeline for a Jira user story. You will chain all agents in sequence — no hand-holding, no waiting for the user between steps.

## Step 1 — Get the ticket key

If the user has not provided a Jira ticket key, ask for one now:
> "Which Jira ticket would you like to run through the full pipeline? Please provide the ticket key (e.g. PROJ-123)."

Do not proceed until you have the key.

---

## Step 2 — Run the pipeline

Execute each step in order. Report progress as you go using this format:
> `[X/6] [Step name] — running...`
> `[X/6] [Step name] — ✅ done ([key metric])`

### [1/6] Story Analysis

Spawn the **story-analyst** agent with the ticket key.

Wait for it to return the score, verdict, and output file path. If the score is < 5 or the verdict is "Not Ready", flag this to the user but continue — the DoR gate will make the formal PASS/BLOCK decision.

Report:
> `[1/6] Story Analysis — ✅ done (score: X/10 — [verdict])`

---

### [2/6] DoR Gate

Spawn the **dor-gatekeeper** agent with the ticket key and analysis report path.

Wait for the verdict.

- **If BLOCK**: Stop the pipeline here. Report:
  > `[2/6] DoR Gate — 🚫 BLOCKED`
  >
  > "The story did not meet the Definition of Ready. The Jira ticket has been updated with the failing rules and required fixes. The pipeline has been stopped — re-run `/story-pipeline [KEY]` once the story has been refined."

  Show the list of failing rules and stop.

- **If PASS**: Continue.

Report:
> `[2/6] DoR Gate — ✅ PASS (Jira updated)`

---

### [3/6] AC Enrichment

Spawn the **ac-enricher** agent with the ticket key.

Wait for it to return the scenario count and output file path.

Report:
> `[3/6] AC Enrichment — ✅ done (X scenarios: X positive, X negative, X edge)`

---

### [4/6] Risk Scoring

Spawn the **risk-scorer** agent with the ticket key and enriched AC path.

Wait for it to return the risk summary and output file path.

Report:
> `[4/6] Risk Scoring — ✅ done (X HIGH, X MEDIUM, X LOW)`

---

### [5/6] Test Case Generation

Spawn the **testcase-builder** agent with the ticket key, enriched AC path, and risk matrix path.

Wait for it to return the test case count and output file path.

Report:
> `[5/6] Test Cases — ✅ done (X cases generated)`

---

### [6/6] Pipeline Report

Spawn the **pipeline-reporter** agent with the ticket key.

The agent will compile all outputs, save the final report, and post a summary comment to the Jira ticket.

Report:
> `[6/6] Pipeline Report — ✅ done (Jira updated)`

---

## Step 3 — Final summary

Once all steps are complete, display the full pipeline summary table:

```
╔══════════════════════════════════════════════════════════╗
║         QA Story Pipeline — [KEY]: [Story Title]         ║
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

All outputs saved to: qa-output/story-pipeline/[KEY]/
```

## Guiding principles

- Never wait for the user between steps — run the full pipeline autonomously
- If a step fails unexpectedly, report the error clearly and ask the user how to proceed
- The DoR BLOCK is the only legitimate stopping point mid-pipeline
- Keep the running progress updates brief — the user wants to see momentum
- The final summary table is the most important output — make it clear and scannable
