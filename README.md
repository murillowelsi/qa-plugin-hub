# qa-plugin-hub

A curated collection of Claude Code plugins for QA and software engineering workflows. Two toolkits covering the full testing lifecycle — from story analysis and manual test design to Playwright E2E automation.

---

## Plugins

- [qa-story-pipeline](#qa-story-pipeline) — automated ISTQB-aligned story pipeline
- [qa-playwright-toolkit](#qa-playwright-toolkit) — Playwright E2E automation toolkit

---

## qa-story-pipeline

An automated pipeline that takes a raw Jira story and produces everything a QA team needs to start testing: analysis report, DoR verdict, enriched BDD scenarios, risk matrix, and prioritised test cases — all posted back to Jira.

### Structure

```
plugins/qa-story-pipeline/
├── .claude-plugin/
│   └── plugin.json          ← name, version, author, keywords
└── skills/
    ├── story-pipeline/      ← 🎯 MAIN ENTRY POINT (run everything)
    ├── story-analyzer/      ← step 1 standalone
    ├── dor-gatekeeper/      ← step 2 standalone
    ├── ac-enricher/         ← step 3 standalone
    ├── risk-scorer/         ← step 4 standalone
    ├── testcase-builder/    ← step 5 standalone
    └── story-refiner/       ← rescue skill (blocked stories)
```

### How it works

```
  USER                   SKILL                        JIRA / DISK
  ────                   ─────                        ───────────

  /story-pipeline KEY
        │
        ▼
  ┌───────────┐
  │  [1/6]    │  fetch ticket ──────────────────────► Jira (read)
  │  Analyze  │  INVEST + AC quality + testability
  │           │  score 0–10, find CRITICALs          ► 01-analysis.md
  └─────┬─────┘
        │
        ▼
  ┌───────────┐
  │  [2/6]    │  apply DoR rules                      ► dor-verdict.json
  │  DoR Gate │  score ≥7, no CRITICALs, ≥2 ACs      ► Jira (comment)
  └─────┬─────┘
        │
     PASS? ──── NO ──► 🚫 BLOCKED
        │                    │
        │              /story-refiner KEY
        │              show rewrite → user approves
        │              → update Jira → re-run pipeline
        │ YES
        ▼
  ┌───────────┐
  │  [3/6]    │  rewrite ACs into Given/When/Then
  │  Enrich   │  + negative paths + edge cases        ► 02-enriched-ac.md
  │  ACs      │
  └─────┬─────┘
        │
        ▼
  ┌───────────┐
  │  [4/6]    │  group into functional areas
  │  Risk     │  likelihood × impact = risk score      ► 03-risk-matrix.md
  │  Scoring  │  HIGH ≥15 | MEDIUM 8–14 | LOW ≤7
  └─────┬─────┘
        │
        ▼
  ┌───────────┐
  │  [5/6]    │  generate test cases
  │  Test     │  HIGH areas first, descriptive titles  ► 04-test-cases.md
  │  Cases    │  no TC-001 IDs, no Gherkin
  └─────┬─────┘
        │
        ▼
  ┌───────────┐
  │  [6/6]    │  compile all metrics                   ► 05-pipeline-report.md
  │  Report   │  post summary ──────────────────────── ► Jira (comment)
  └───────────┘
```

### Output files

Each pipeline run produces a folder per ticket:

```
qa-output/story-pipeline/<KEY>/
├── 01-analysis.md          score, findings by severity
├── dor-verdict.json        PASS / BLOCK + failing rules
├── 02-enriched-ac.md       Given/When/Then scenarios (gherkin blocks)
├── 03-risk-matrix.md       ranked risk areas
├── 04-test-cases.md        prioritised test cases
├── 05-pipeline-report.md   full summary
└── refined-story.md        (only if /story-refiner was run)
```

### Skills

Each skill is self-contained. Run individually or let `story-pipeline` chain them all.

| Skill | Description |
|---|---|
| `story-pipeline` | Runs the full pipeline end-to-end — all 6 steps, no hand-holding |
| `story-analyzer` | ISTQB analysis: INVEST criteria, AC quality, testability, completeness. Score 0–10 |
| `dor-gatekeeper` | Enforces Definition of Ready. Posts PASS or BLOCK verdict to Jira |
| `ac-enricher` | Rewrites ACs into Given/When/Then BDD scenarios with negative paths and edge cases |
| `risk-scorer` | Scores functional areas by likelihood × impact. Produces a ranked risk matrix |
| `testcase-builder` | Generates structured test cases ordered by risk priority |
| `story-refiner` | Rewrites a blocked story to meet DoR. Shows rewrite for review before updating Jira |

### Usage

```
# Run the full pipeline
/story-pipeline PROJ-123

# Run individual steps
/story-analyzer PROJ-123
/dor-gatekeeper PROJ-123
/ac-enricher PROJ-123
/risk-scorer PROJ-123
/testcase-builder PROJ-123

# Fix a blocked story
/story-refiner PROJ-123
```

**Typical flow for a blocked story:**
```
/story-pipeline PROJ-123    → 🚫 BLOCKED at DoR gate
/story-refiner PROJ-123     → review rewrite → approve → Jira updated
/story-pipeline PROJ-123    → ✅ full pipeline completes
```

---

## qa-playwright-toolkit

A complete Playwright automation toolkit covering project setup, test planning, test generation, and test maintenance.

### Skills

| Skill | Description |
|---|---|
| `project-init` | Scaffolds a new Playwright project with TypeScript, Page Object Model structure, and a ready-to-run config |
| `test-planner` | Explores a live app in a real browser, produces a structured test plan, and optionally creates a Jira ticket and feature branch |
| `test-generator` | Generates Playwright test files in TypeScript using the Page Object Model from a test plan |
| `test-healer` | Debugs and fixes failing Playwright tests, diagnosing root causes from error output |

### Usage

```
# Explore a live app and produce a test plan
/test-planner https://your-app.com

# Initialize a new Playwright project
/project-init

# Generate tests from a plan
/test-generator

# Fix failing tests
/test-healer
```

---

## Installation

### Install the full hub

```bash
claude plugin add https://raw.githubusercontent.com/murillowelsi/qa-plugin-hub/main/.claude-plugin/marketplace.json
```

### Install a single plugin

```bash
# Story pipeline only
claude plugin add https://raw.githubusercontent.com/murillowelsi/qa-plugin-hub/main/.claude-plugin/marketplace.json --plugin qa-story-pipeline

# Playwright toolkit only
claude plugin add https://raw.githubusercontent.com/murillowelsi/qa-plugin-hub/main/.claude-plugin/marketplace.json --plugin qa-playwright-toolkit
```

### Install from a local clone

```bash
git clone https://github.com/murillowelsi/qa-plugin-hub.git
claude plugin add ./qa-plugin-hub/.claude-plugin/marketplace.json
```

---

## Prerequisites

- [Claude Code](https://claude.ai/code) CLI installed
- **Jira integration** (`story-pipeline`, `story-analyzer`, `dor-gatekeeper`, `story-refiner`): Atlassian Rovo MCP configured in Claude Code settings
- **Browser-based skills** (`test-planner`): Playwright MCP configured in Claude Code settings

---

## License

MIT — see individual plugin directories for details.

## Author

[Murillo Welsi](mailto:murillo.welsi@gmail.com)
