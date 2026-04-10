# qa-plugin-hub

A curated collection of Claude Code plugins for software engineering and QA workflows. This hub provides two toolkits that cover the full testing lifecycle — from story analysis and manual test design to Playwright E2E automation.

## Plugins

### qa-story-pipeline

An automated ISTQB-aligned story pipeline that chains story analysis, Definition of Ready enforcement, AC enrichment, risk scoring, and test case generation — all end-to-end without manual hand-holding.

| Skill | Description |
|---|---|
| `story-pipeline` | Runs the full automated pipeline end-to-end: story analysis → DoR gate → AC enrichment → risk scoring → test case generation → Jira report |
| `analyze-story` | Analyzes a Jira user story using ISTQB quality checks (INVEST, clarity, testability, AC quality) and produces a severity-rated report |
| `dor-gatekeeper` | Enforces the Definition of Ready: reads the story analysis, applies DoR rules, and posts a PASS or BLOCK verdict to Jira |
| `ac-enricher` | Rewrites raw acceptance criteria into structured Given/When/Then BDD scenarios, adding edge cases and negative paths |
| `risk-scorer` | Scores each functional area by likelihood of failure × business impact and produces a ranked risk matrix |
| `testcase-builder` | Generates structured test cases from enriched ACs and risk scores, covering positive, negative, boundary, and edge scenarios ordered by risk priority |

### qa-playwright-toolkit

A complete Playwright automation toolkit covering project setup, test planning, test generation, and test maintenance.

| Skill | Description |
|---|---|
| `project-init` | Scaffolds a new Playwright project with TypeScript, Page Object Model structure, and a ready-to-run config |
| `test-planner` | Explores a live app in a real browser, produces a structured test plan, and optionally creates a Jira ticket and feature branch |
| `test-generator` | Generates Playwright test files in TypeScript using the Page Object Model from a test plan |
| `test-healer` | Debugs and fixes failing Playwright tests, diagnosing root causes from error output |

---

## Installation

This hub is distributed as a [Claude Code plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces#create-the-marketplace-file). You can install individual plugins or the entire hub.

### Install the full hub

Point Claude Code at this repository's marketplace file:

```bash
claude plugin add https://raw.githubusercontent.com/murillowelsi/qa-plugin-hub/main/.claude-plugin/marketplace.json
```

### Install a single plugin

Install only the story pipeline toolkit:

```bash
claude plugin add https://raw.githubusercontent.com/murillowelsi/qa-plugin-hub/main/.claude-plugin/marketplace.json --plugin qa-story-pipeline
```

Install only the Playwright toolkit:

```bash
claude plugin add https://raw.githubusercontent.com/murillowelsi/qa-plugin-hub/main/.claude-plugin/marketplace.json --plugin qa-playwright-toolkit
```

### Install from a local clone

```bash
git clone https://github.com/murillowelsi/qa-plugin-hub.git
claude plugin add ./qa-plugin-hub/.claude-plugin/marketplace.json
```

---

## Usage

Once installed, skills are available as slash commands inside any Claude Code session.

### Run the full QA pipeline for a Jira ticket

```
/story-pipeline PROJ-123
```

This chains story analysis, DoR enforcement, AC enrichment, risk scoring, and test case generation — all without manual hand-holding. A final report is posted back to Jira.

### Analyze a user story

```
/analyze-story PROJ-123
```

### Check Definition of Ready

```
/dor-gatekeeper PROJ-123
```

### Enrich acceptance criteria into BDD scenarios

```
/ac-enricher PROJ-123
```

### Score and rank risk

```
/risk-scorer
```

### Generate test cases

```
/testcase-builder
```

### Explore a live app and produce a Playwright test plan

```
/test-planner https://your-app.com
```

### Initialize a new Playwright project

```
/project-init
```

### Fix failing Playwright tests

```
/test-healer
```

---

## Prerequisites

- [Claude Code](https://claude.ai/code) CLI installed
- For Jira integration: Atlassian Rovo MCP configured in your Claude Code settings
- For browser-based skills (`test-planner`): Playwright MCP configured in your Claude Code settings

---

## License

MIT — see individual plugin directories for details.

## Author

[Murillo Welsi](mailto:murillo.welsi@gmail.com)
