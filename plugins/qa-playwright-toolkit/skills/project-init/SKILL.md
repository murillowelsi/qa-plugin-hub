---
name: project-init
description: >
  Initialize a new Playwright project from scratch following best practices: TypeScript, Page Object Model (POM) folder structure inside an e2e/ directory (e2e/pages/, e2e/fixtures/, e2e/tests/), baseURL in e2e/playwright.config.ts, and a ready-to-use split fixture scaffold using mergeTests.
  Use this skill whenever the user wants to start a new Playwright project, set up Playwright in an existing repo, scaffold a test framework, or says something like "init playwright", "set up playwright", "create a playwright project", "npm init playwright", or "start playwright testing from scratch".
---

# Playwright Project Init

You are setting up a new Playwright project from scratch. Your goal is to run the official scaffolder, then reshape the output into a clean, maintainable structure ready for Page Object Model tests.

This skill is **website agnostic** — it creates empty scaffolding only. No pages, no test files, no app-specific fixtures, no credentials. Everything application-specific is added later.

## Step 1 — Run the scaffolder

Run the official init command in the `e2e/` subdirectory (create it if it doesn't exist). Keeping Playwright inside `e2e/` isolates it from the main project's dependencies and keeps root-level `package.json` clean.

```bash
mkdir -p e2e && cd e2e && npm init playwright@latest .
```

Recommended answers:
- **TypeScript or JavaScript?** → TypeScript
- **Where to put your end-to-end tests?** → `tests`
- **Add a GitHub Actions workflow?** → `false` (unless the user explicitly wants CI)
- **Install Playwright browsers?** → `true`

## Step 2 — Clean up scaffolded examples

```bash
rm -f e2e/tests/example.spec.ts
rm -rf e2e/tests-examples
```

## Step 3 — Set up the POM folder structure

Create this directory layout inside `e2e/`:

```
e2e/
├── core/
│   └── constants.ts          ← single process.env access point
├── fixtures/
│   ├── index.ts              ← mergeTests of all fixture files + re-exports expect
│   └── pages.ts              ← POM fixtures (empty scaffold)
├── pages/                    ← empty; Page Object classes added later
├── tests/                    ← empty; test files added later
└── playwright.config.ts
```

Create the empty directories:

```bash
mkdir -p e2e/core e2e/fixtures e2e/pages
```

### `core/constants.ts`

Single place where `process.env` is accessed. Add variables here as the project grows — never read `process.env` directly in fixtures or tests.

```ts
// Add required env vars here as needed. Example:
// function requireEnv(name: string): string {
//   const value = process.env[name];
//   if (!value) throw new Error(`Missing required environment variable: ${name}`);
//   return value;
// }

export const BASE_URL = process.env.BASE_URL ?? 'http://localhost:8080';
```

### `fixtures/pages.ts`

Empty scaffold — add one fixture per Page Object class as they are created:

```ts
import { test as base } from '@playwright/test';

// Add page fixtures here as you create Page Object classes in pages/
// Example:
// import { SomePage } from '../pages/SomePage';
// type PageFixtures = { somePage: SomePage };
// export const test = base.extend<PageFixtures>({
//   somePage: async ({ page }, use) => { await use(new SomePage(page)); },
// });

export const test = base;
```

### `fixtures/index.ts`

Merge all fixture files here. Specs always import from this file, never from `@playwright/test` directly:

```ts
import { mergeTests, expect } from '@playwright/test';
import { test as pageFixtures } from './pages';

export const test = mergeTests(pageFixtures);
export { expect };
```

When new fixture files are added (e.g. `auth.ts`, `api.ts`), import and merge them here.

## Step 4 — Configure playwright.config.ts

Replace the scaffolded config entirely. Use an **absolute path** for `dotenv` — paths resolved relative to `__dirname` are stable regardless of which directory `npm run test:e2e` is run from:

```ts
import { defineConfig, devices } from '@playwright/test';
import * as dotenv from 'dotenv';
import path from 'path';

dotenv.config({ path: path.resolve(__dirname, '.env') });

export default defineConfig({
  testDir: './tests',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  outputDir: './test-results',
  reporter: [['html', { outputFolder: './playwright-report' }]],
  use: {
    baseURL: process.env.BASE_URL ?? 'http://localhost:8080',
    trace: 'on-first-retry',
  },
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
  ],
});
```

Ask the user for the `baseURL` if it wasn't provided. Setting it here means all `page.goto()` calls and `toHaveURL()` assertions use relative paths — never hardcode the full origin.

Add a `test:e2e` script to the root `package.json` so tests can be run from the project root:

```json
{
  "scripts": {
    "test:e2e": "playwright test --config e2e/playwright.config.ts"
  }
}
```

## Step 5 — Create .env and update .gitignore

Create `e2e/.env`:

```
BASE_URL=http://localhost:8080
```

Add to the root `.gitignore`:

```
/e2e/.env
/e2e/test-results/
/e2e/playwright-report/
/e2e/playwright/.cache/
```

## Step 6 — Verify the setup

```bash
npx playwright test --config e2e/playwright.config.ts --list
```

No tests yet is fine — `--list` should exit cleanly. Fix any errors before finishing.

## What NOT to do

- Don't generate any test files — `tests/` must stay empty. Tests are added later via `/playwright-test-generator`.
- Don't create any Page Object classes in `pages/` — that directory must stay empty.
- Don't add app-specific fixtures (auth, credentials, API resources) — this scaffold is website agnostic.
- Don't install or configure ESLint.
- Don't use `dotenv.config()` without an absolute path — it resolves from CWD, which breaks when `npm run test:e2e` is run from the project root instead of `e2e/`.
- Don't hardcode absolute URLs in `goto()` calls — `baseURL` in the config handles the origin.
- Don't import `test` or `expect` directly from `@playwright/test` in spec files — always import from `'../fixtures'`.
- Don't put `pages/` or `fixtures/` inside `tests/` — keep them at the `e2e/` root.

## Summary for the user

After setup, tell the user:

1. The project is ready. `pages/` is where Page Object classes go, `fixtures/pages.ts` is where they get wired up as fixtures.
2. All spec files import `test` and `expect` from `'../fixtures'`, never from `@playwright/test` directly.
3. `baseURL` is set — use relative paths everywhere (`page.goto('/')` not `page.goto('https://...')`).
4. As the project grows, add fixture files (e.g. `fixtures/auth.ts`, `fixtures/api.ts`) and merge them into `fixtures/index.ts`.
5. Suggest running `/playwright-test-planner` next to explore the app and generate a test plan, then `/playwright-test-generator` to implement it.
