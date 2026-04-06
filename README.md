# qa-plugin-hub

A curated collection of Claude Code plugins for software engineering and QA workflows. This hub provides two toolkits that cover the full testing lifecycle — from story analysis and manual test design to Playwright E2E automation.

## Plugins

### qa-manual-toolkit

A complete manual QA toolkit that guides you from story refinement to a prioritized, ready-to-execute test suite.

| Skill | Description |
|---|---|
| `analyze-story` | Analyzes a Jira user story using ISTQB quality checks (INVEST, clarity, testability, AC quality) and produces a severity-rated report |
| `generate-test-cases` | Derives positive, negative, edge case, and non-functional test cases from acceptance criteria |
| `assess-risk` | Scores and ranks test cases by likelihood × impact to produce a phased execution order |
| `explore-and-plan` | Explores a live web application in a real browser and produces a structured manual test plan |
| `report-bug` | Structures a bug report from a failed test case and creates a Jira ticket with full reproduction steps |
| `qa-orchestrator` | Autonomous pipeline that chains all phases end-to-end (story analysis → test generation → risk assessment → optional Playwright automation) for a given Jira ticket |

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

Install only the manual QA toolkit:

```bash
claude plugin add https://raw.githubusercontent.com/murillowelsi/qa-plugin-hub/main/.claude-plugin/marketplace.json --plugin qa-manual-toolkit
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
/qa-orchestrator PROJ-123
```

This chains story analysis, test case generation, risk assessment, and optionally Playwright test generation — all without manual hand-holding.

### Analyze a user story

```
/analyze-story PROJ-123
```

### Generate test cases

```
/generate-test-cases PROJ-123
```

### Assess risk and prioritize

```
/assess-risk
```

### Explore a live app and produce a test plan

```
/explore-and-plan https://your-app.com
```

### Plan and generate Playwright tests

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

### File a bug report to Jira

```
/report-bug
```

---

## Prerequisites

- [Claude Code](https://claude.ai/code) CLI installed
- For Jira integration: Atlassian Rovo MCP configured in your Claude Code settings
- For browser-based skills (`explore-and-plan`, `test-planner`): Playwright MCP configured in your Claude Code settings

---

## License

MIT — see individual plugin directories for details.

## Author

[Murillo Welsi](mailto:murillo.welsi@gmail.com)
