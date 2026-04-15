---
name: test-planner
description: Create a comprehensive Playwright test plan for a web app or website by exploring it live in a browser, then optionally create a Jira ticket, create a feature branch, and generate the tests. Use this skill whenever the user wants to plan tests, create a test plan, explore an app for QA coverage, or figure out what scenarios to automate — even if they don't say "test plan" explicitly. Triggers on phrases like "plan tests for", "what should we test", "create test scenarios for", "explore and plan", or any URL/app that needs test coverage mapped out. Works with or without Jira.
allowed-tools: Bash, Read, Write, Edit, mcp__*__getJiraIssue, mcp__*__createJiraIssue, mcp__*__addCommentToJiraIssue, mcp__*__getVisibleJiraProjects, mcp__*__*
---

# Playwright Test Planner

You are an expert QA engineer. Your job is to explore a live web application using browser tools, produce a comprehensive structured test plan, and kick off test generation. Jira integration and branch creation are optional — proceed based on what the user asked for.

## Workflow

### Step 0 — Set up the browser session

Before using any browser tool, call `planner_setup_page` to initialize the browser session. Do NOT pass a `seedFile` — let it auto-generate:

```
planner_setup_page()
```

If this fails with a message like "did not expect test() to be called here" or "two different versions of @playwright/test", the project has a module-instance conflict. Fix it before continuing:

1. Open `playwright.config.ts` — change `from '@playwright/test'` to `from 'playwright/test'`
2. Open `fixtures/index.ts` — do the same
3. Open `.mcp.json` — change `"command": "npx"` / `"args": ["playwright", ...]` to:
   ```json
   {
     "command": "./node_modules/.bin/playwright",
     "args": ["run-test-mcp-server"]
   }
   ```
4. Ask the user to reload the MCP server (restart Claude Code or run `/mcp` to reconnect), then retry `planner_setup_page`.

This conflict happens when `npx playwright` resolves to a globally-cached version that is a different module instance from the project's local `@playwright/test`. Using `./node_modules/.bin/playwright` pins the MCP server to the exact local copy.

### Step 1 — Explore the app

Use `browser_navigate`, `browser_snapshot`, `browser_click`, `browser_hover`, `browser_type`, and other browser tools to discover all interactive elements, forms, navigation paths, and dynamic behaviors. Snapshots are cheaper than screenshots — prefer them.

### Step 2 — Map user flows

Identify primary journeys (happy paths), edge cases, error states, and boundary conditions. Consider different user types.

### Step 3 — Write the test plan

Produce a markdown test plan in this structure:

```markdown
# <App Name> - Comprehensive Test Plan

## Application Overview
<Brief description of the app and its key sections>

## Test Scenarios

### 1. <Feature Area>

#### 1.1 <Scenario Title>

**File:** `tests/<scenario-name>.spec.ts`

**Steps:**
1. <Action>
   - expect: <Assertion>
2. <Action>
   - expect: <Assertion>
```

Then:
- Call `planner_save_plan` with the complete markdown.
- Save the file to `qa-output/test-plans/<app-name>-test-plan.md`.

### Step 4 — Jira ticket (optional)

After saving the test plan, ask the user: **"Would you like me to create a Jira ticket for this test plan?"**

Wait for their answer before proceeding. If they say no (or anything equivalent), skip to Step 5. If they say yes, continue below.

Use `murillowelsi.atlassian.net` as the `cloudId` for all Atlassian MCP calls. If a call fails with a permissions error, fall back to `mcp__atlassian__getAccessibleAtlassianResources` to find the right site.

**Resolve the project key:** If the user didn't specify a Jira project, call `mcp__atlassian__getVisibleJiraProjects` to list available projects and pick the most relevant one (or ask the user if ambiguous).

Create a Jira Story:
```
mcp__atlassian__createJiraIssue
  cloudId: <resolved>
  projectKey: <project key>
  summary: "[Test Plan] <App Name> — Automated Test Coverage"
  issueType: Story
  description: <full test plan markdown>
  contentFormat: markdown
```

After creation, note the ticket key (e.g., `SCRUM-42`). Also post the plan as a comment:
```
mcp__atlassian__addCommentToJiraIssue
  cloudId: <resolved>
  issueIdOrKey: <ticket key>
  commentBody: <full test plan markdown>
  contentFormat: markdown
```

### Step 5 — Create a feature branch (optional)

**Skip this step** if no Jira ticket was created and the user didn't explicitly ask for a branch.

**Run this step** if a ticket was created (use the ticket key in the branch name) or if the user asked to create a branch.

Branch naming:
- With ticket: `feature/<TICKET-KEY>-<kebab-case-app-name>-playwright-tests`
- Without ticket: `feature/<kebab-case-app-name>-playwright-tests`

Before branching, ensure you're on the default branch:
```bash
git checkout main 2>/dev/null || git checkout master 2>/dev/null || true
```

If the repo has no commits yet, create an initial empty commit first:
```bash
git commit --allow-empty -m "chore: init"
```

Then create and checkout the branch:
```bash
git checkout -b feature/<branch-name>
```

Tell the user the branch that was created.

### Step 6 — Generate the tests

Invoke the `playwright-test-generator` skill. Pass it the path to the saved test plan file (`qa-output/test-plans/<app-name>-test-plan.md`) so it starts from the right plan without needing to re-explore the app.

---

## Quality standards

- Each scenario must be **independent** — assume a blank/fresh browser state.
- Steps must be specific enough for the test generator to automate without guessing.
- Always include:
  - Happy path scenarios
  - Negative/error scenarios (invalid input, empty states, validation messages)
  - Edge cases and boundary conditions
- Expected outcomes must be concrete and verifiable (exact text, URL, element visibility).
- Group scenarios by feature area with numbered headings.
- Assign a unique output file path per scenario under `tests/`.

## Classifying scenarios: read-only vs. mutating

When writing scenarios, flag each one as **read-only** (just navigates/asserts) or **mutating** (creates, edits, or deletes data). This distinction matters for test generation:

- **Read-only** scenarios can share seeded/static data and use `test.beforeEach` for navigation.
- **Mutating** scenarios must use API-driven fixture setup and teardown (`testProject`, etc. in `fixtures/scenarios.ts`) — never rely on UI-created data left over from a previous test.

Also note: **never use `test.beforeEach` for navigation in tests that use a scenario fixture** (e.g. `testProject`). Playwright initialises fixtures lazily, so a fixture declared only in the test body is set up *after* `beforeEach` runs. The page loads before the data exists. The test generator handles this by calling `goto()` as the first line of each mutating test body instead.
