# QA Issue Pipeline — SDLC Integration

## Overview

The QA Issue Pipeline is an automated quality gate that runs **before development begins**. It enforces the shift-left testing principle: find and fix quality problems at the requirements stage, where the cost of change is lowest.

It takes a raw Jira story and produces everything a QA team needs to start testing — analysis report, DoR verdict, enriched BDD scenarios, risk matrix, and prioritised test cases — all posted back to Jira without manual intervention.

---

## Where It Sits in the SDLC

```
IDEA ──▶ BACKLOG ──▶ REFINEMENT ──▶ SPRINT ──▶ DEV ──▶ TEST ──▶ RELEASE
                          ▲
                          │
             Pipeline runs HERE — before a single
             line of code is written.
```

Traditional QA catches bugs late, during or after development. The pipeline moves quality analysis as far left as possible — catching ambiguous requirements, missing acceptance criteria, and untestable stories during refinement, not during a sprint.

**Cost of a bug by phase (industry rule of thumb):**

| Phase found       | Relative cost to fix |
|-------------------|----------------------|
| Requirements      | 1x                   |
| Development       | 5x                   |
| Testing           | 10x                  |
| Production        | 50x+                 |

---

## Pipeline Steps and SDLC Mapping

```
SDLC PHASE          PIPELINE STEP           PURPOSE
─────────────────────────────────────────────────────────────────────────

BACKLOG             ┌─────────────────┐
GROOMING     ──────▶│ 1. STORY        │  Scores the issue across INVEST,
                    │    ANALYSIS     │  clarity, AC quality, testability,
                    └────────┬────────┘  and completeness. Flags gaps
                             │           before the team wastes effort.
                             ▼
DEFINITION          ┌─────────────────┐
OF READY     ──────▶│ 2. DoR GATE     │  Enforces entry criteria to sprint.
                    │                 │  BLOCKS stories that aren't ready.
                    └────────┬────────┘  Posts verdict to Jira automatically.
                             │
                      PASS? ─┤─ BLOCK? ──▶ stop + notify + suggest fix
                             │
                             ▼
SPRINT              ┌─────────────────┐
PLANNING     ──────▶│ 3. AC           │  Transforms vague ACs into structured
                    │    ENRICHMENT   │  Given/When/Then BDD scenarios with
                    └────────┬────────┘  negative paths and edge cases.
                             │
                             ▼
SPRINT              ┌─────────────────┐
PLANNING     ──────▶│ 4. RISK         │  Scores each functional area by
                    │    SCORING      │  likelihood × business impact.
                    └────────┬────────┘  Tells the team WHERE to focus first.
                             │
                             ▼
DEV / QA            ┌─────────────────┐
EXECUTION    ──────▶│ 5. TEST CASE    │  Generates ready-to-run test cases
                    │    GENERATION   │  ordered by risk — HIGH risk areas
                    └────────┬────────┘  always tested first.
                             │
                             ▼
SPRINT REVIEW       ┌─────────────────┐
/ REPORTING  ──────▶│ 6. PIPELINE     │  Summarises all outputs and posts
                    │    REPORT       │  a full report to Jira.
                    └─────────────────┘
```

---

## Step Details

### Step 1 — Story Analysis

Evaluates the story against ISTQB-aligned quality dimensions:

| Dimension             | What it checks |
|-----------------------|----------------|
| INVEST criteria       | Independent, Negotiable, Valuable, Estimable, Small, Testable |
| Title clarity         | Understandable to any team member; no technical scope tags |
| Description clarity   | Clear user role, goal, and rationale |
| AC quality            | Specific, measurable, unambiguous; covers the main path |
| Testability           | Test cases derivable from ACs without guessing |
| Completeness          | No missing ACs, description, story points, or designs |

Each finding is rated CRITICAL / HIGH / MEDIUM / LOW. A weighted score (0–10) is produced.

**Verdict thresholds:**

| Score | Verdict           |
|-------|-------------------|
| ≥ 8   | Ready             |
| 5–7   | Needs Refinement  |
| < 5   | Not Ready         |

### Step 2 — Definition of Ready Gate

Enforces hard entry rules before a story can enter a sprint:

| Rule                  | Requirement |
|-----------------------|-------------|
| Score threshold       | ≥ 7/10 |
| No CRITICAL findings  | Zero CRITICALs allowed |
| Minimum ACs           | At least 2 clear acceptance criteria |
| Estimable             | Enough information to size the story |
| Clear description     | Description present and understandable |

A BLOCK stops the pipeline and posts a detailed comment to Jira explaining exactly what needs to be fixed. The `/issue-refiner` skill can automatically rewrite the story to resolve all findings.

### Step 3 — AC Enrichment

Rewrites each acceptance criterion into full BDD (Behaviour-Driven Development) scenarios:

- **Positive path** — the happy path as defined in the original AC
- **Negative path** — wrong, missing, or invalid input
- **Edge case** — boundary values, empty states, concurrency, system limits

This output is shared with developers and product so the whole team aligns on expected behaviour before implementation.

### Step 4 — Risk Scoring

Groups scenarios into functional areas and scores each by:

```
Risk Score = Likelihood of Failure (1–5) × Business Impact (1–5)

HIGH   ≥ 15
MEDIUM  8–14
LOW    ≤ 7
```

#### How risk is estimated

Risk scoring currently uses **generic ISTQB heuristics** — it does not have access to the system's codebase, historical bug data, or architecture. It relies on:

- **Domain keyword signals** — words like `payment`, `auth`, `cancel`, `notification` carry implied risk weight based on common industry failure patterns
- **AC surface area** — more scenarios means more complexity and higher likelihood of failure
- **Failure pattern heuristics** — e.g. cancellation flows have complex state transitions; email notifications depend on external services

**Limitation:** this produces a useful starting point, not ground truth. The team should challenge and adjust scores based on:

- Known fragile areas in the codebase
- Past incidents in this domain
- Integrations known to be unreliable
- Regulatory or compliance weight not captured in the ticket

The risk matrix is a **forcing function** — it ensures risk is explicitly discussed during refinement rather than skipped.

### Step 5 — Test Case Generation

Produces execution-ready test cases ordered by risk priority — HIGH first, then MEDIUM, then LOW. Each test case includes preconditions, numbered steps, and expected results. No Gherkin syntax — plain test cases any tester can run.

### Step 6 — Pipeline Report

Aggregates all outputs into a single summary and posts it to Jira so the full team has visibility without opening individual files.

---

## Artefacts Produced

All outputs are saved to `qa-output/issue-pipeline/<KEY>/`:

| File                  | Contents |
|-----------------------|----------|
| `01-analysis.md`      | Story quality report with scored findings |
| `dor-verdict.json`    | Machine-readable PASS / BLOCK verdict |
| `02-enriched-ac.md`   | BDD scenarios (positive + negative + edge) |
| `03-risk-matrix.md`   | Functional areas ranked by risk score |
| `04-test-cases.md`    | Execution-ready test cases ordered by priority |
| `05-pipeline-report.md` | Full summary; mirrors the Jira comment |

---

## Running the Pipeline

```bash
# Full pipeline (recommended)
/issue-pipeline PROJ-123

# Individual steps (standalone)
/issue-analyzer PROJ-123
/dor-gatekeeper PROJ-123
/ac-enricher PROJ-123
/risk-scorer PROJ-123
/testcase-builder PROJ-123

# If DoR blocks — rewrite the story automatically
/issue-refiner PROJ-123
```

---

## Design Principles

- **Shift-left** — quality analysis happens before development, not after
- **Autonomous** — the full pipeline runs without user hand-holding between steps
- **Jira-native** — all verdicts and reports are posted as Jira comments; no separate tool to check
- **DoR as a hard gate** — a BLOCK is the only legitimate stopping point mid-pipeline
- **Risk-ordered output** — every output that can be prioritised is, so testing effort is never wasted on low-risk areas while high-risk ones go untested
