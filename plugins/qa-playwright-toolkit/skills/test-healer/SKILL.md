---
name: test-healer
description: Debug and fix failing Playwright tests. Use this skill whenever tests are failing, broken, or producing errors — even if the user just says "tests are broken", "fix the tests", "something's failing", or pastes an error message from a Playwright run. Always use this skill to heal tests rather than rewriting them from scratch.
---

# Playwright Test Healer

You are an expert Playwright automation engineer specializing in debugging and fixing broken tests. Work systematically: diagnose first, fix second, verify last. Never rewrite tests wholesale — make targeted fixes.

## Workflow

1. **Run all tests** — call `test_run` to get the full failure picture. Note which tests fail and their error messages.
2. **Debug each failing test** — call `test_debug` for each failure. The test will pause at the error.
3. **Investigate the failure** — while paused, use browser tools to understand the current page state:
   - `browser_snapshot` — see what's actually on the page
   - `browser_generate_locator` — find the correct selector for an element
   - `browser_console_messages` — check for JS errors
   - `browser_network_requests` — check for failed API calls
   - `browser_evaluate` — run arbitrary JS to inspect state
4. **Identify root cause** — categorize the failure:
   - Selector changed (element renamed/restructured)
   - Timing issue (element not yet visible/enabled)
   - Assertion mismatch (text/value changed)
   - Data dependency (test assumes state that doesn't exist)
   - Application regression (the app itself changed behavior)
5. **Fix the code** — edit only what's broken. Prefer:
   - Updating selectors in the Page Object (`tests/pages/`) rather than in the spec file
   - Using `getByRole` / `getByLabel` / `getByText` — never XPath, never positional `.nth(n)`, never auto-generated CSS strings
   - Scoping ambiguous locators to a unique parent via `.filter({ has: ... })` or a semantic container locator
   - Removing explicit waits — rely on Playwright's auto-waiting
6. **Verify the fix** — call `test_run` again (or re-run just the fixed test). Confirm it passes cleanly.
7. **Repeat** until all tests pass.

## Fixing guidelines

**Selectors that changed**: Update the locator in the POM class, not in the spec. If the element is dynamic, use `getByRole` with a name regex for resilience.

**Timing issues**: Replace `waitForTimeout` with `waitFor({ state: 'visible' })` or a retrying assertion like `await expect(locator).toBeVisible()`.

**Assertion mismatches**: Check whether the expected value is stale (e.g., a hardcoded string that changed). Use regex for dynamic content.

**Flaky tests**: Look for race conditions. Add `await expect(locator).toBeVisible()` before interactions. Never use `networkidle`.

## POM-aware fixes

This project uses the Page Object Model. Before editing a spec file, check if the broken locator lives in `pages/`. Fix it there — that way all tests using the POM benefit.

```
├── fixtures/
│   ├── index.ts          ← mergeTests of all fixture files — import test/expect from here
│   ├── pages.ts          ← POM fixtures
│   ├── auth.ts           ← credentials fixtures
│   ├── api.ts            ← api fixture { projects, tasks, ... }
│   └── scenarios.ts      ← scenario data fixtures (testProject, etc.)
├── requests/
│   ├── ProjectsRequests.ts
│   └── TasksRequests.ts
├── pages/                ← fix locators here first
├── tests/auth.setup.ts   ← login once, saves playwright/.auth/user.json
└── tests/*.spec.ts       ← fix assertions and flow here
```

## Data-dependency failures — lazy fixture ordering

A common root cause for **"element not found" / timeout** errors in tests that use scenario fixtures (e.g. `testProject`) is fixture lazy-init ordering:

- Playwright initialises a fixture only when first accessed.
- If `testProject` is used only in the test body (not in `beforeEach`), it is set up **after** `beforeEach` runs.
- A `beforeEach` that calls `page.goto()` will load the page **before** the fixture data exists, so the row never appears.

**Fix**: remove the navigation from `beforeEach` and call `goto()` as the first line of the test body.

```ts
// ✅ Correct
test('should edit', async ({ projectsPage, testProject }) => {
  await projectsPage.goto(); // runs after testProject is created
  await projectsPage.clickProjectRow(testProject.name);
});

// ❌ Wrong — beforeEach navigates before testProject fixture runs
test.beforeEach(async ({ projectsPage }) => { await projectsPage.goto(); });
test('should edit', async ({ projectsPage, testProject }) => {
  await projectsPage.clickProjectRow(testProject.name); // row doesn't exist yet
});
```

## Scenario teardown — data pollution

If a test fails because the same text/row appears multiple times (strict mode violation), the likely cause is leftover data from previous runs. The correct fix is **not** `.first()` — it is ensuring teardown runs via a scenario fixture:

- Scenario fixtures (`testProject`, etc.) live in `fixtures/scenarios.ts`.
- They create data via API before the test and delete it (plus any child resources) after, even if the test itself fails.
- Tests that create data through the UI (e.g. "should create a project") must clean up via `api` at the end of the test body — they cannot use a pre-built scenario fixture.

```ts
// Teardown pattern in scenarios.ts
testProject: async ({ api }, use) => {
  const project = await api.projects.create({ ... });
  await use(project);
  // cleanup child resources first, then parent — 404-tolerant
  const tasks = await api.tasks.list();
  await Promise.all(
    tasks.filter(t => t.projectId === project.id)
         .map(t => api.tasks.delete(t.id).catch(() => {}))
  );
  await api.projects.delete(project.id).catch(() => {});
},
```

## Authentication state issues

The project uses a `setup` project that runs `tests/auth.setup.ts` once and saves session state to `playwright/.auth/user.json`. All tests load this state automatically via `storageState` in `playwright.config.ts`.

If tests fail with redirect-to-login, 401, or "not authenticated" errors:
1. Check whether `playwright/.auth/user.json` exists — if missing, the setup project hasn't run yet.
2. Re-run with `npx playwright test` (not just the failing spec) so the setup project runs first.
3. If `auth.setup.ts` itself is failing, debug the login flow — locators on the login form may have changed.
4. If the session is expired, delete `playwright/.auth/user.json` and re-run so a fresh session is saved.

## baseURL and navigation

`playwright.config.ts` sets a `baseURL` for the project. Before fixing navigation-related failures, read `playwright.config.ts` to find the actual `baseURL`. Always use relative paths — never introduce absolute URLs when fixing tests.

- `goto()` in POMs: `await this.page.goto('/articles')` not the full URL
- `page.goto()` in specs: `await navPage.page.goto('/')` not the full URL
- `toHaveURL()`: `await expect(page).toHaveURL('/dashboard')` not the full URL

If you encounter a test using absolute URLs, fix it to use relative paths.

## Last resort: mark as fixme

If a test is failing due to a genuine application bug (not a test bug), and you've confirmed this with the browser tools, mark it as skipped with an explanation:

```ts
test.fixme('scenario name', async ({ page }) => {
  ...
});
```

Add a `fixme` annotation with the reason instead of inline comments:

```ts
test.fixme('scenario name');
```

Or use `test.info().annotations.push` if you need to record the reason programmatically.

Only use `test.fixme` when you are confident the test logic is correct and the app behavior is wrong. Document exactly what's happening vs. what should happen.

## What never to do

- Never use XPath — no `locator('xpath=...')`, no `locator('xpath=preceding-sibling::...')`, no XPath in any form.
- Never use `waitForTimeout` or `networkidle`.
- Never use positional `.nth(n)` without a semantic anchor.
- Never use long auto-generated CSS strings (e.g., `.animate-scroll > div:nth-child(17) > ...`).
- Never skip a test with `test.skip` — use `test.fixme` with a comment if it must be bypassed.
- Never ask the user for input — make the most reasonable fix and verify it.
- Never rewrite a test from scratch when a targeted fix will do.
