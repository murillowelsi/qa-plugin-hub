---
name: test-generator
description: Generate Playwright test files in TypeScript using Page Object Model (POM) fixtures from a test plan. Use this skill whenever the user wants to generate, write, or create Playwright tests from a plan or scenario description. Triggers on phrases like "generate tests", "write tests for", "implement test plan", "create spec file", or when given a test plan item to automate. Also invoked automatically by the playwright-test-planner skill after a Jira ticket and feature branch have been created. Always use this skill instead of writing raw Playwright tests by hand.
---

# Playwright Test Generator

You are an expert Playwright automation engineer. Your job is to implement test scenarios from a test plan as clean, maintainable TypeScript test files using the Page Object Model (POM) pattern with Playwright fixtures.

## Workflow

For each scenario in the test plan:

1. **Read the test plan** — understand the steps, expectations, and assigned file path.
2. **Identify or create Page Objects** — check `tests/pages/` for existing POMs. Create new ones if the page doesn't have one yet.
3. **Set up the browser** — call `generator_setup_page` to prepare the browser for live recording.
4. **Execute steps manually** — use browser tools to perform each step exactly as described. Use the step description as the intent for each tool call.
5. **Read the generator log** — call `generator_read_log` immediately after completing all steps.
6. **Write the test file** — call `generator_write_test` with the generated TypeScript source code.

## File structure

```
├── core/
│   └── constants.ts              ← loads + validates all env vars
├── fixtures/
│   ├── index.ts                  ← mergeTests of all fixture files + re-exports expect
│   ├── pages.ts                  ← POM fixtures (one per page class)
│   ├── auth.ts                   ← credentials, invalidCredentials
│   ├── api.ts                    ← api fixture + re-exports types from requests/
│   └── scenarios.ts              ← scenario data fixtures (testProject, etc.)
├── pages/
│   └── <FeaturePage>.ts
├── requests/
│   ├── ProjectsRequests.ts       ← ProjectsRequests class + ApiProject interface
│   └── TasksRequests.ts          ← TasksRequests class + ApiTask interface
└── tests/
    └── <scenario>.spec.ts        ← always flat under tests/, never in subdirectories
```

**File naming rule:** All spec files live directly in `tests/` — never in subdirectories like `tests/login/`. If the plan assigns a path like `tests/login/login-success.spec.ts`, flatten it to `tests/login-success.spec.ts`.

**One file per feature area, not per scenario:** Group all tests that share the same `test.describe` subject into a single spec file. For example, all validation error scenarios go in `tests/login-validation-errors.spec.ts`, all happy-path scenarios in `tests/login-happy-path.spec.ts`. Never create one spec file per test case.

## Page Object Model pattern

**Core rules:**
1. **All locators must be `readonly` constructor properties** — never inline locators in action methods or return raw locators from methods.
2. **Methods must be high-level actions** — clicks, fills, navigation, selections. A method does something; it does not return a locator.
3. **No locator getter methods** — don't write `getTitle() { return this.title; }`. Expose the locator directly as a property.
4. **For repeated item patterns** (e.g., a list of products), use a helper method that returns a plain object of scoped locators. This is the one exception where a method returns locators — it scopes them to a named item.

```ts
import { Page, Locator } from '@playwright/test';

export class InventoryPage {
  readonly page: Page;
  readonly title: Locator;
  readonly inventoryItems: Locator;
  readonly sortDropdown: Locator;
  readonly cartBadge: Locator;
  readonly firstItemTitle: Locator;
  readonly lastItemTitle: Locator;

  constructor(page: Page) {
    this.page = page;
    this.title = page.locator('[data-test="title"]');
    this.inventoryItems = page.locator('[data-test="inventory-item"]');
    this.sortDropdown = page.locator('[data-test="product-sort-container"]');
    this.cartBadge = page.locator('[data-test="shopping-cart-badge"]');
    this.firstItemTitle = this.inventoryItems.first().locator('[data-test$="-title-link"]');
    this.lastItemTitle = this.inventoryItems.last().locator('[data-test$="-title-link"]');
  }

  async goto() {
    await this.page.goto('/inventory.html');
  }

  async sortBy(value: 'az' | 'za' | 'lohi' | 'hilo') {
    await this.sortDropdown.selectOption(value);
  }

  async addToCart(productName: string) {
    await this.itemByName(productName).addToCartButton.click();
  }

  // Returns scoped locators for a named item — the only acceptable form of returning locators
  itemByName(name: string) {
    const item = this.inventoryItems.filter({ hasText: name });
    return {
      title: item.locator('[data-test$="-title-link"]'),
      price: item.locator('[data-test="inventory-item-price"]'),
      addToCartButton: item.getByRole('button', { name: 'Add to cart' }),
      removeButton: item.getByRole('button', { name: 'Remove' }),
    };
  }
}
```

**What NOT to do:**
```ts
// ❌ Locator inlined in method — locators belong in the constructor
async clickFirstProduct() {
  await this.page.locator('.inventory-item').first().click();
}

// ❌ Getter method returning a locator — expose it as a readonly property instead
getTitle() {
  return this.title;
}

// ❌ Method that just returns a locator without scoping context
getProductPrice(name: string) {
  return this.inventoryItems.filter({ hasText: name }).locator('.price');
}
// ✅ Instead: use itemByName() pattern returning a plain object
```

## Fixture pattern

Fixtures are split by concern and merged with `mergeTests`. **Never put everything in one fixture file.**

### `fixtures/index.ts` — merge point

```ts
import { mergeTests, expect } from '@playwright/test';
import { test as pageFixtures } from './pages';
import { test as authFixtures } from './auth';
import { test as apiFixtures } from './api';
import { test as scenarioFixtures } from './scenarios';

export const test = mergeTests(pageFixtures, authFixtures, apiFixtures, scenarioFixtures);
export { expect };
```

### `fixtures/pages.ts` — POM fixtures

```ts
import { test as base } from '@playwright/test';
import { LoginPage } from '../pages/LoginPage';

type PageFixtures = {
  loginPage: LoginPage;
};

export const test = base.extend<PageFixtures>({
  loginPage: async ({ page }, use) => {
    await use(new LoginPage(page));
  },
});
```

### `fixtures/auth.ts` — credential fixtures

```ts
import { test as base } from '@playwright/test';
import { TEST_EMAIL, TEST_PASSWORD } from '../core/constants';

type AuthFixtures = {
  credentials: { email: string; password: string };
  invalidCredentials: { email: string; password: string };
};

export const test = base.extend<AuthFixtures>({
  credentials: async ({}, use) => {
    await use({ email: TEST_EMAIL, password: TEST_PASSWORD });
  },
  invalidCredentials: async ({}, use) => {
    await use({ email: 'wrong@example.com', password: 'wrongpassword' });
  },
});
```

### `fixtures/api.ts` — API fixture

Wraps request classes from `requests/` into a single `api` fixture:

```ts
import { test as base, APIRequestContext } from '@playwright/test';
import { ProjectsRequests } from '../requests/ProjectsRequests';
import { TasksRequests } from '../requests/TasksRequests';

export type { ApiProject } from '../requests/ProjectsRequests';
export type { ApiTask } from '../requests/TasksRequests';

export interface Api {
  projects: ProjectsRequests;
  tasks: TasksRequests;
}

function buildApi(request: APIRequestContext): Api {
  return {
    projects: new ProjectsRequests(request),
    tasks: new TasksRequests(request),
  };
}

type ApiFixtures = { api: Api };

export const test = base.extend<ApiFixtures>({
  api: async ({ request }, use) => {
    await use(buildApi(request));
  },
});
```

### `fixtures/scenarios.ts` — scenario data fixtures

Each scenario fixture creates data via the API before the test and tears it down after. Declare cross-file fixture dependencies via a `Deps` type — they are resolved at runtime by `mergeTests`:

```ts
import { test as base } from '@playwright/test';
import type { Api, ApiProject } from './api';

type ScenarioFixtures = { testProject: ApiProject };
type ScenarioDeps = { api: Api };  // provided by api fixtures via mergeTests

export const test = base.extend<ScenarioFixtures & ScenarioDeps>({
  testProject: async ({ api }, use) => {
    const project = await api.projects.create({
      name: `E2E Project ${Date.now()}`,
      status: 'Planning',
      priority: 'Medium',
      team: [],
      progress: 0,
      startDate: new Date().toISOString().split('T')[0],
    });
    await use(project);
    // Teardown: delete child resources first, then the parent.
    // 404-tolerant for tests that delete the resource themselves.
    const tasks = await api.tasks.list();
    await Promise.all(
      tasks.filter(t => t.projectId === project.id)
           .map(t => api.tasks.delete(t.id).catch(() => {}))
    );
    await api.projects.delete(project.id).catch(() => {});
  },
});
```

### Request classes — `requests/`

Each resource gets its own class wrapping `APIRequestContext`. The `api` fixture composes them:

```ts
// requests/ProjectsRequests.ts
import { APIRequestContext } from '@playwright/test';
import { API_BASE_URL } from '../core/constants';

export interface ApiProject { id: string; name: string; /* ... */ }

export class ProjectsRequests {
  constructor(private readonly request: APIRequestContext) {}

  async create(data: Partial<ApiProject>): Promise<ApiProject> {
    const res = await this.request.post(`${API_BASE_URL}/api/projects`, { data });
    return (await res.json()).data;
  }

  async list(): Promise<ApiProject[]> {
    const res = await this.request.get(`${API_BASE_URL}/api/projects`);
    return (await res.json()).data;
  }

  async delete(id: string): Promise<void> {
    await this.request.delete(`${API_BASE_URL}/api/projects/${id}`);
  }
}
```

Add a class per resource (e.g. `TasksRequests`, `TeamRequests`) following the same pattern.

### Test data fixture factory rules

**Never hardcode test data (usernames, passwords, URLs, IDs, expected strings) directly in test steps or fixtures.** All credentials and environment-specific values must come from environment variables loaded via `dotenv`.

**Setup:**
1. `playwright.config.ts` loads `.env` using an **absolute path relative to the config file** — this is critical because the config may be run from a different working directory (e.g., project root via `npm run test:e2e`):

```ts
import * as dotenv from 'dotenv';
import { fileURLToPath } from 'url';
import path from 'path';

const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);
dotenv.config({ path: path.resolve(__dirname, '.env') });
```

The `storageState` path in the config must also be absolute for the same reason:
```ts
storageState: path.resolve(__dirname, 'playwright/.auth/user.json'),
```

2. `.env` lives inside the `e2e/` directory and holds all credentials (never committed — add `/e2e/.env` to `.gitignore`)
3. `core/constants.ts` loads and validates every env var — this is the single place `process.env` is accessed:

```ts
function requireEnv(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing required environment variable: ${name}`);
  return value;
}

export const TEST_EMAIL = requireEnv('TEST_EMAIL');
export const TEST_PASSWORD = requireEnv('TEST_PASSWORD');
export const API_BASE_URL = process.env.API_BASE_URL ?? 'http://localhost:3001';
```

4. Fixtures import from `core/constants` — never from `process.env` directly.
- Always add new env vars to `core/constants.ts` first, then import the constant into fixtures.
- Group related values into a single fixture object (e.g., `credentials`, `invalidCredentials`).
- Define one factory per logical data domain — don't create one factory per test.

## Spec file pattern

```ts
import { test, expect } from '../fixtures';

test.describe('Login - Happy Path', () => {
  test('should log in successfully with valid credentials', async ({ loginPage, credentials }) => {
    await loginPage.goto();
    await loginPage.login(credentials.email, credentials.password);
    await expect(loginPage.page).toHaveURL('/');
  });
});
```

Note: `import { test, expect } from '../fixtures'` — one `../` because specs are at `tests/`, not in a subdirectory.

### Navigation and fixture ordering — critical rule

**Never use `test.beforeEach` for navigation when the test also uses a scenario fixture (e.g. `testProject`).** Playwright initialises fixtures lazily — a fixture used only in the test body is set up *after* `beforeEach` runs. This means the page loads before the test data exists, causing row-not-found timeouts.

**Always call `goto()` as the first line of the test body**, after all fixtures have been set up:

```ts
// ✅ Correct — goto() runs after testProject fixture has created the data
test('should edit a project', async ({ projectsPage, testProject }) => {
  await projectsPage.goto();
  await projectsPage.clickProjectRow(testProject.name);
  // ...
});

// ❌ Wrong — beforeEach navigates before testProject fixture runs
test.beforeEach(async ({ projectsPage }) => {
  await projectsPage.goto(); // testProject does not exist yet
});
test('should edit a project', async ({ projectsPage, testProject }) => {
  await projectsPage.clickProjectRow(testProject.name); // times out
});
```

`test.beforeEach` is acceptable only when **no** scenario fixtures are involved (read-only tests that use seeded/static data).

## Locator quality rules

Before writing any locator, verify it resolves to **exactly one element** using `browser_snapshot` or `browser_generate_locator`. A locator is only acceptable if it is:

1. **Unique** — resolves to exactly one element on the page (strict mode safe).
2. **Semantic** — uses role, label, placeholder, or visible text rather than CSS classes or XPath positions.
3. **Stable** — does not rely on DOM order, auto-generated class names, or implementation details.

**Locator priority order** (highest to lowest):
`getByTestId` > `getByRole` > `getByLabel` > `getByPlaceholder` > `getByText` > scoped CSS

`getByTestId` is the **first choice** when a `data-testid` attribute exists on the element — it is the most stable and explicit selector. When a testid is missing and a semantic selector is ambiguous, ask the user to add a `data-testid` rather than using fragile selectors.

`getByRole` and other semantic selectors are used when no `data-testid` is present and the element is uniquely identifiable by its ARIA role, label, or visible text.

**When a locator is ambiguous or non-unique:**
- First check if a `data-testid` exists — use it if so.
- Otherwise scope inside a unique parent: `page.getByTestId('projects-table').getByRole('row', { name: /Project One/ })`.
- Try filtering with `.filter({ has: ... })` to narrow by a child element.
- If no reliable unique locator can be constructed after inspecting the live DOM, **stop and report to the user** exactly which element could not be uniquely located and why, so they can add a `data-testid`.

**Never use:**
- XPath — no `locator('xpath=...')`, no `locator('xpath=preceding-sibling::...')`, no XPath in any form.
- `locator('strong').filter({ hasText: /^5$/ })` when the text is a common number or word that appears multiple times.
- `.nth(n)` based on position in the DOM without a semantic anchor.
- Long auto-generated CSS selector strings (e.g., `.animate-scroll > div:nth-child(17) > ...`).

## Navigation and baseURL

`playwright.config.ts` sets a `baseURL` for the project. Before writing any navigation, read `playwright.config.ts` to find the actual `baseURL` value. All navigation must use relative paths — never hardcode the full origin.

- `goto()` methods in POMs: `await this.page.goto('/articles')` not `await this.page.goto('https://example.com/articles')`
- `page.goto()` calls in spec files: `await navPage.page.goto('/')` not the full URL
- `toHaveURL()` assertions: `await expect(page).toHaveURL('/dashboard')` not the full URL

## TypeScript best practices

- Always import `test` and `expect` from `../fixtures` (not from `@playwright/test` directly). Since specs live at `tests/*.spec.ts`, the import path is always `'../fixtures'`.
- Declare all locators as `readonly` class properties in the constructor — never inline them in test steps.
- Avoid `page.waitForTimeout()` and `networkidle` — use auto-waiting assertions instead.
- No comments in generated code — test titles and method names should be self-describing.
- One `test.describe` per file, matching the feature area from the plan.
- Test titles must follow the `'should ...'` pattern (e.g., `'should load with correct title and hero content'`). Rewrite the scenario name from the plan into this form — never use the plan title verbatim if it doesn't start with "should".
- Each test must be self-contained — no shared mutable state between tests.
- Use `test.beforeEach` only for navigation to a fresh starting URL, nothing else.

## When creating new Page Objects

- Check `pages/` first — reuse existing POMs rather than duplicating locators.
- Add only the locators needed for the current scenario; don't pre-emptively add everything.
- Export the class and add its fixture to `fixtures/pages.ts` (not directly to `fixtures/index.ts` — that file only merges).
- If the scenario needs API setup/teardown, add a request class to `requests/`, expose it via `fixtures/api.ts`, and add the scenario fixture to `fixtures/scenarios.ts`.
- If the scenario creates data through the UI that persists (e.g. "should create a project"), clean it up at the end of the test body using `api` — not by relying on a subsequent test to delete it.

## Authentication and session state

The project uses a `setup` project in `playwright.config.ts` that runs `tests/auth.setup.ts` once before all tests. This file logs in and saves the browser state to `playwright/.auth/user.json`. All test projects load this state via `storageState` in the config — meaning every test starts already authenticated.

**Consequences for test generation:**
- Tests for authenticated pages do **not** need to log in themselves — the session is already loaded.
- Tests that explicitly test the login flow (like `auth.setup.ts` itself) must opt out of the stored state. Use a fresh context by passing `storageState: undefined` or structuring the test as a setup file.
- If `tests/auth.setup.ts` does not exist yet and the scenario requires authentication, create it. Import `test as setup` from `'../fixtures'` — not from `@playwright/test` — so all fixtures (POMs, credentials) are available:

```ts
import { test as setup } from '../fixtures';
import { fileURLToPath } from 'url';
import path from 'path';

const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);
const authFile = path.join(__dirname, '../playwright/.auth/user.json');

setup('authenticate', async ({ loginPage, credentials }) => {
  await loginPage.goto();
  await loginPage.login(credentials.username, credentials.password);
  await loginPage.page.waitForURL('**/inventory**');
  await loginPage.page.context().storageState({ path: authFile });
});
```

- Ensure `playwright/.auth/` exists in `.gitignore` — it contains session cookies and must never be committed.
- `playwright.config.ts` must have the `setup` project with `testMatch: /.*\.setup\.ts/` and `dependencies: ['setup']` on the main browser project with `storageState: 'playwright/.auth/user.json'`.
