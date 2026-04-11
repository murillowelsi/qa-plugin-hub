# qa-plugin-hub

A curated collection of Claude Code plugins for QA and software engineering workflows. Two toolkits covering the full testing lifecycle — from story analysis and manual test design to Playwright E2E automation.

---

## How the two plugins connect

```
  SDLC PHASE          PLUGIN                        SKILL
  ─────────────────────────────────────────────────────────────────────────

  BACKLOG             ┌──────────────────────┐
  GROOMING     ──────▶│  qa-issue-pipeline   │  /issue-pipeline
                      │                      │  /issue-analyzer
                      │  Analyzes story,     │  /dor-gatekeeper
                      │  enforces DoR,       │  /ac-enricher
                      │  enriches ACs,       │  /risk-scorer
                      │  scores risk,        │  /testcase-builder
                      │  generates           │  /issue-refiner
                      │  test cases          │  /bug-reporter
                      └──────────┬───────────┘
                                 │
                           PASS? ┤ BLOCK? ──▶ /issue-refiner ──▶ re-run
                                 │
  ─────────────────────────────────────────── handoff ────────────────────
                                 │
  SPRINT               ┌─────────▼────────────┐
  PLANNING      ──────▶│  qa-playwright-      │  /automation-planner
                        │  toolkit             │
                        │                      │  Triages test cases:
                        │  Decides what to     │  AUTOMATE / API TEST /
                        │  automate, scaffolds │  MANUAL / SKIP
                        │  specs, heals        │
                        │  broken tests,       │  /project-init
                        │  audits coverage     │  /test-planner
                        └──────────┬───────────┘  /test-generator
                                   │              /test-healer
  DEV / CI               [specs]   │              /coverage-auditor
                    ───────────────┘
                                   │
                    ┌──────────────▼────────────┐
                    │  playwright test (CI/CD)   │
                    │  runs on every PR / merge  │
                    └──────────────┬────────────┘
                                   │
                          pass? ───┤─── fail? ──▶ /test-healer
                                   │
  RELEASE              ┌───────────▼────────────┐
                  ─────▶  /coverage-auditor      │  Are HIGH risk test cases
                        │                        │  actually covered by specs?
                        └────────────────────────┘
```

---

## Plugins

- [qa-issue-pipeline](#qa-issue-pipeline) — automated ISTQB-aligned issue pipeline
- [qa-playwright-toolkit](#qa-playwright-toolkit) — Playwright E2E automation toolkit

---

## qa-issue-pipeline

An automated pipeline that takes a raw Jira story and produces everything a QA team needs to start testing: analysis report, DoR verdict, enriched BDD scenarios, risk matrix, and prioritised test cases — all posted back to Jira.

### Structure

```
plugins/qa-issue-pipeline/
├── .claude-plugin/
│   └── plugin.json          ← name, version, author, keywords
└── skills/
    ├── issue-pipeline/      ← 🎯 MAIN ENTRY POINT (run everything)
    ├── issue-analyzer/      ← step 1 standalone
    ├── dor-gatekeeper/      ← step 2 standalone
    ├── ac-enricher/         ← step 3 standalone
    ├── risk-scorer/         ← step 4 standalone
    ├── testcase-builder/    ← step 5 standalone
    ├── issue-refiner/       ← rescue skill (blocked stories)
    └── bug-reporter/        ← defect documentation → Jira
```

### How it works

```
  USER                   SKILL                        JIRA / DISK
  ────                   ─────                        ───────────

  /issue-pipeline KEY
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
        │              /issue-refiner KEY
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
qa-output/issue-pipeline/<KEY>/
├── 01-analysis.md          score, findings by severity
├── dor-verdict.json        PASS / BLOCK + failing rules
├── 02-enriched-ac.md       Given/When/Then scenarios (gherkin blocks)
├── 03-risk-matrix.md       ranked risk areas
├── 04-test-cases.md        prioritised test cases
├── 05-pipeline-report.md   full summary
└── refined-story.md        (only if /issue-refiner was run)
```

### Skills

Each skill is self-contained. Run individually or let `issue-pipeline` chain them all.

| Skill | Description |
|---|---|
| `issue-pipeline` | Runs the full pipeline end-to-end — all 6 steps, no hand-holding |
| `issue-analyzer` | ISTQB analysis: INVEST criteria, AC quality, testability, completeness. Score 0–10 |
| `dor-gatekeeper` | Enforces Definition of Ready. Posts PASS or BLOCK verdict to Jira |
| `ac-enricher` | Rewrites ACs into Given/When/Then BDD scenarios with negative paths and edge cases |
| `risk-scorer` | Scores functional areas by likelihood × impact. Produces a ranked risk matrix |
| `testcase-builder` | Generates structured test cases ordered by risk priority |
| `issue-refiner` | Rewrites a blocked story to meet DoR. Shows rewrite for review before updating Jira |
| `bug-reporter` | Creates a structured bug report from a description or Jira ticket, posts to Jira as comment or new issue |

### Usage

```
# Run the full pipeline
/issue-pipeline PROJ-123

# Run individual steps
/issue-analyzer PROJ-123
/dor-gatekeeper PROJ-123
/ac-enricher PROJ-123
/risk-scorer PROJ-123
/testcase-builder PROJ-123

# Fix a blocked story
/issue-refiner PROJ-123

# Write a bug report
/bug-reporter PROJ-123
/bug-reporter  # or describe the bug in free text
```

**Typical flow for a blocked story:**
```
/issue-pipeline PROJ-123    → 🚫 BLOCKED at DoR gate
/issue-refiner PROJ-123     → review rewrite → approve → Jira updated
/issue-pipeline PROJ-123    → ✅ full pipeline completes
```

---

## qa-playwright-toolkit

A complete Playwright automation toolkit covering automation planning, project setup, test planning, test generation, test maintenance, and coverage auditing. Connects directly with `qa-issue-pipeline` outputs to close the loop from story to spec.

### How it connects to the pipeline

```
  /issue-pipeline output           /qa-playwright-toolkit input
  ────────────────────────         ──────────────────────────────
  04-test-cases.md       ────────▶  /automation-planner  (what to automate)
  03-risk-matrix.md      ────────▶  /automation-planner  (priority order)
  02-enriched-ac.md      ────────▶  /test-generator      (scenario structure)
  automation-plan.md     ────────▶  /test-generator      (triage decisions)
```

### Skills

| Skill | Description |
|---|---|
| `automation-planner` | Fetches the Jira ticket + pipeline outputs and triages every test case: AUTOMATE / API TEST / MANUAL / SKIP. The handoff between pipeline and specs |
| `project-init` | Scaffolds a new Playwright project with TypeScript, Page Object Model structure, and a ready-to-run config |
| `test-planner` | Explores a live app in a real browser, produces a structured test plan, and optionally creates a Jira ticket and feature branch |
| `test-generator` | Generates Playwright test files in TypeScript using the Page Object Model from a test plan or automation plan |
| `test-healer` | Debugs and fixes failing Playwright tests, diagnosing root causes from error output |
| `coverage-auditor` | Cross-references all pipeline test cases against existing spec files to surface HIGH risk gaps and score coverage by risk level |

### Usage

```
# Triage pipeline test cases into what to automate
/automation-planner PROJ-123

# Initialize a new Playwright project
/project-init

# Explore a live app and produce a test plan
/test-planner https://your-app.com

# Generate tests from a plan or automation plan
/test-generator

# Fix failing tests
/test-healer

# Audit automation coverage across all pipeline tickets
/coverage-auditor

# Audit coverage for a specific ticket
/coverage-auditor PROJ-123
```

**Typical end-to-end flow:**
```
/issue-pipeline PROJ-123      → pipeline outputs in qa-output/issue-pipeline/PROJ-123/
/automation-planner PROJ-123  → automation-plan.md: what to automate and why
/test-generator               → Playwright specs scaffolded from automation plan
/coverage-auditor PROJ-123    → verify HIGH risk gaps are covered
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
claude plugin add https://raw.githubusercontent.com/murillowelsi/qa-plugin-hub/main/.claude-plugin/marketplace.json --plugin qa-issue-pipeline

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
- **Jira integration** (`issue-pipeline`, `issue-analyzer`, `dor-gatekeeper`, `issue-refiner`, `automation-planner`): Atlassian Rovo MCP configured in Claude Code settings
- **Browser-based skills** (`test-planner`): Playwright MCP configured in Claude Code settings

---

## License

MIT — see individual plugin directories for details.

## Author

[Murillo Welsi](https://github.com/murillowelsi)
