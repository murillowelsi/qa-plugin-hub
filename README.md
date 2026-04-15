# qa-plugin-hub

Two Claude Code plugins covering the full testing lifecycle — from raw Jira story to Playwright spec.

```
  BACKLOG / GROOMING                          SPRINT / DEV / CI
  ──────────────────                          ─────────────────

  qa-issue-pipeline          handoff          qa-playwright-toolkit
  ─────────────────    ─────────────────►    ─────────────────────
  /issue-pipeline            test cases       /automation-planner
  /sprint-quality-gate       risk matrix      /test-generator
  /issue-analyzer            enriched ACs     /test-planner
  /dor-gatekeeper                             /project-init
  /ac-enricher                                /test-healer
  /risk-scorer                                /coverage-auditor
  /testcase-builder
  /issue-refiner
  /ticket-splitter
  /bug-reporter
```

---

## qa-issue-pipeline

Takes a raw Jira story and produces everything a QA team needs to start testing: ISTQB analysis, DoR verdict, enriched BDD scenarios, risk matrix, and prioritised test cases — all posted back to Jira.

### Pipeline flow

```
                         +------------------+
                         |   Jira Ticket    |
                         |      (KEY)       |
                         +--------+---------+
                                  |
                                  v
                         +------------------+
                         | Issue Type Guard |
                         +--------+---------+
                                  |
               +------------------+------------------+
               |                                     |
           NOT OK                                   OK
               |                             Story / Bug / Task
               v                                     |
   +-----------------------+                         v
   | Type Violation        |              +---------------------+
   |                       |              |   [1/6] ANALYSIS    |
   | Spike  --> Task       |              |                     |
   | Subtask-> Parent      |              | 8 ISTQB dimensions  |
   | Epic   -> Stories     |              | Score: 0-10         |
   | Other  -> nearest     |              | 01-analysis.md      |
   +-----------------------+              +----------+----------+
                                                     |
                                                     v
                                          +---------------------+
                                          |   [2/6] DoR GATE    |
                                          |                     |
                                          | score >= 7?         |
                                          | zero CRITICALs?     |
                                          | >= 2 ACs?           |
                                          | estimable?          |
                                          | clear description?  |
                                          | work item struct?   |
                                          +----------+----------+
                                                     |
                              +----------------------+----------------------+
                              |                                             |
                            BLOCK                                         PASS
                              |                                             |
                              v                                             v
              +-------------------------------+               +------------------------+
              | Structural issue?             |               | Post checkmark to Jira |
              |                               |               +------------+-----------+
              |  YES --> /ticket-splitter     |                            |
              |                               |                            v
              |  NO  --> /issue-refiner       |          +-----------------+-----------------+
              |          (offer inline)       |          |     [3+4/6] PARALLEL AGENTS       |
              |               |               |          |                                   |
              |          Jira updated?        |          |  Agent A              Agent B     |
              |               |               |          |  AC ENRICHMENT        RISK SCORE  |
              |          restart pipeline?    |          |                                   |
              |               |               |          |  Given/When/Then      L x I score |
              +---------------+---------------+          |  + negative path      HIGH >= 15  |
                        restart ^                        |  + edge cases         MED  8-14   |
                                |                        |                       LOW  <= 7   |
                                |                        |  02-enriched-ac.md    03-risk-    |
                                |                        |                         matrix.md |
                                |                        +--------+----------+---------------+
                                |                                 |          |
                                |                                 +----+-----+
                                |                                      |
                                |                                      v
                                |                        +---------------------+
                                |                        | [5/6] TEST CASES    |
                                |                        |                     |
                                |                        | Ordered by risk:    |
                                |                        | HIGH --> MED --> LOW|
                                |                        |                     |
                                |                        | Positive / Negative |
                                |                        | Boundary / Edge     |
                                |                        |                     |
                                |                        | 04-test-cases.md    |
                                |                        +----------+----------+
                                |                                   |
                                |                                   v
                                |                        +---------------------+
                                |                        | [6/6] REPORT        |
                                |                        |                     |
                                |                        | 05-pipeline-        |
                                |                        |   report.md         |
                                |                        | Jira comment        |
                                |                        | (user approves)     |
                                |                        +----------+----------+
                                |                                   |
                                |                                   v
                                |                        +---------------------+
                                |                        | PIPELINE COMPLETE   |
                                |                        |                     |
                                |                        | qa-output/          |
                                |                        | issue-pipeline/KEY/ |
                                |                        |   01-analysis.md    |
                                |                        |   02-enriched-ac.md |
                                |                        |   03-risk-matrix.md |
                                |                        |   04-test-cases.md  |
                                |                        |   05-pipeline-      |
                                |                        |      report.md      |
                                |                        |   dor-verdict.json  |
                                |                        |   refined-story.md  |
                                |                        +---------------------+
                                |
                       offer to restart
                       after refiner


    ======================= STANDALONE SKILLS ===========================

    +------------------+   +------------------+   +--------------------+
    | /bug-reporter    |   | /ticket-splitter  |   | /sprint-quality-   |
    |                  |   |                  |   |       gate         |
    | Free text or     |   | Too large?       |   |                    |
    | Jira key         |   | Multi-deliverable|   | Sprint name/ID     |
    |       |          |   | Tech-layer frag? |   |        |           |
    |       v          |   |        |         |   |        v           |
    | Production?      |   |        v         |   | All tickets in     |
    | Yes --> Bug      |   | Split proposal   |   | sprint, parallel:  |
    | No  --> Task     |   | User approves    |   |                    |
    |       |          |   | Create in Jira   |   | Story    --> full  |
    |       v          |   |                  |   |            pipeline|
    | Structured       |   +------------------+   | Bug/Task --> QA    |
    | report           |                          |             check  |
    | Post to Jira     |                          | Other    --> type  |
    +------------------+                          |           violation|
                                                  |        |           |
                                                  |        v           |
                                                  | sprint-report.md   |
                                                  | + per-ticket Jira  |
                                                  |   comments         |
                                                  +--------------------+
```

### Skills

| Skill | When to use |
|---|---|
| `issue-pipeline` | Run the full 6-step pipeline for a single ticket |
| `sprint-quality-gate` | Run the pipeline across every ticket in a sprint in parallel |
| `issue-analyzer` | ISTQB analysis only — INVEST, AC quality, testability, score 0–10 |
| `dor-gatekeeper` | Enforce Definition of Ready, post PASS / BLOCK verdict to Jira |
| `ac-enricher` | Rewrite ACs into Given/When/Then with negative paths and edge cases |
| `risk-scorer` | Score functional areas by likelihood × impact, produce ranked risk matrix |
| `testcase-builder` | Generate structured test cases ordered by risk priority |
| `issue-refiner` | Rewrite a blocked story to meet DoR, apply to Jira after review |
| `ticket-splitter` | Decompose oversized or fragmented tickets into independently releasable stories |
| `bug-reporter` | Create a structured bug report from a description or ticket, post to Jira |

### Usage

```bash
# Full pipeline — single ticket
/issue-pipeline PROJ-123

# Full pipeline — entire sprint
/sprint-quality-gate "Sprint 12"

# Run steps individually
/issue-analyzer PROJ-123
/dor-gatekeeper PROJ-123
/ac-enricher PROJ-123
/risk-scorer PROJ-123
/testcase-builder PROJ-123

# Fix a blocked story
/issue-refiner PROJ-123

# Split an oversized ticket
/ticket-splitter PROJ-123

# Document a defect
/bug-reporter PROJ-123
/bug-reporter   # or just describe the bug in plain text
```

**Blocked story:**
```
/issue-pipeline PROJ-123   → BLOCKED at DoR gate
                           → offers to run issue-refiner inline
                           → approve rewrite → restart pipeline
```

**Oversized story:**
```
/issue-pipeline PROJ-123   → flags Work Item Structure (dimension 8)
/ticket-splitter PROJ-123  → split proposal → approve → tickets created in Jira
```

**Sprint readiness check:**
```
/sprint-quality-gate "Sprint 12"   → all tickets in parallel → sprint-report.md
```

---

## qa-playwright-toolkit

Playwright automation toolkit that picks up where `qa-issue-pipeline` leaves off. Takes the pipeline outputs (test cases, risk matrix, enriched ACs) and turns them into real specs.

### How it connects

```
  qa-issue-pipeline output              qa-playwright-toolkit input
  ────────────────────────              ───────────────────────────
  04-test-cases.md       ──────────►   /automation-planner  (triage: automate vs manual)
  03-risk-matrix.md      ──────────►   /automation-planner  (priority order)
  02-enriched-ac.md      ──────────►   /test-generator      (scenario structure)
  automation-plan.md     ──────────►   /test-generator      (triage decisions)
```

### Skills

| Skill | When to use |
|---|---|
| `automation-planner` | Triage every test case: AUTOMATE / API TEST / MANUAL / SKIP |
| `project-init` | Scaffold a new Playwright project with TypeScript and Page Object Model |
| `test-planner` | Explore a live app in a real browser and produce a structured test plan |
| `test-generator` | Generate Playwright specs from a test plan or automation plan |
| `test-healer` | Debug and fix failing Playwright tests |
| `coverage-auditor` | Cross-reference pipeline test cases against specs, surface HIGH risk gaps |

### Usage

```bash
# Triage pipeline output into what to automate
/automation-planner PROJ-123

# Scaffold a new Playwright project
/project-init

# Explore a live app and produce a test plan
/test-planner https://your-app.com

# Generate specs from a plan
/test-generator

# Fix failing tests
/test-healer

# Audit coverage
/coverage-auditor PROJ-123
```

**End-to-end flow:**
```
/issue-pipeline PROJ-123      → outputs in qa-output/issue-pipeline/PROJ-123/
/automation-planner PROJ-123  → automation-plan.md with triage decisions
/test-generator               → Playwright specs scaffolded from the plan
/coverage-auditor PROJ-123    → confirm HIGH risk areas are covered
```

---

## Installation

```bash
# Full hub
claude plugin add https://raw.githubusercontent.com/murillowelsi/qa-plugin-hub/main/.claude-plugin/marketplace.json

# Single plugin
claude plugin add https://raw.githubusercontent.com/murillowelsi/qa-plugin-hub/main/.claude-plugin/marketplace.json --plugin qa-issue-pipeline
claude plugin add https://raw.githubusercontent.com/murillowelsi/qa-plugin-hub/main/.claude-plugin/marketplace.json --plugin qa-playwright-toolkit

# From a local clone
git clone https://github.com/murillowelsi/qa-plugin-hub.git
claude plugin add ./qa-plugin-hub/.claude-plugin/marketplace.json
```

## Prerequisites

- [Claude Code](https://claude.ai/code) CLI
- **Jira MCP** (Atlassian Rovo) — required for all `qa-issue-pipeline` skills
- **Playwright MCP** — required for `/test-planner`

---

## License

MIT — [Murillo Welsi](https://github.com/murillowelsi)
