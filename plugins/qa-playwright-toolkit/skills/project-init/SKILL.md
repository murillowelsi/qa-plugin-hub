---
name: project-init
description: >
  Initialize a new Playwright project from scratch following best practices: TypeScript, Page Object Model (POM) folder structure inside an e2e/ directory (e2e/pages/, e2e/fixtures/, e2e/tests/), baseURL in e2e/playwright.config.ts, and a ready-to-use split fixture scaffold using mergeTests.
  Use this skill whenever the user wants to start a new Playwright project, set up Playwright in an existing repo, scaffold a test framework, or says something like "init playwright", "set up playwright", "create a playwright project", "npm init playwright", or "start playwright testing from scratch".
---

# Playwright Project Init

You are setting up a new Playwright project from scratch. Your goal is to run the official scaffolder, then reshape the output into a clean, maintainable structure ready for Page Object Model tests.

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
│   └── constants.ts          ← loads + validates all env vars; single process.env access point
├── fixtures/
│   ├── index.ts              ← mergeTests of all fixture files + re-exports expect
│   ├── pages.ts              ← POM fixtures
│   ├── auth.ts               ← credentials, invalidCredentials
│   ├── api.ts                ← api fixture + re-exports types from requests/
│   └── scenarios.ts          ← scenario data fixtures (e.g. testProject)
├── pages/
│   └── <FeaturePage>.ts
├── requests/
│   ├── <Resource>Requests.ts ← one class per API resource
│   └── ...
├── playwright/
│   └── .auth/                ← session state, created at runtime, gitignored
├── tests/
│   ├── auth.setup.ts         ← login once, saves playwright/.auth/user.json
│   └── *.spec.ts
└── playwright.config.ts
```

### `core/constants.ts`

Single place where `process.env` is accessed. Throws early with a clear message if a required variable is missing:

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

Fixtures always import from `core/constants` — never from `process.env` directly.

### Split fixture files

Fixtures are split by concern and merged with `mergeTests`. Never put everything in one file.

**`fixtures/pages.ts`** — one fixture per POM:

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

**`fixtures/auth.ts`** — credential fixtures:

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

**`fixtures/api.ts`** — wraps request classes into a single `api` fixture:

```ts
import { test as base, APIRequestContext } from '@playwright/test';
import { ProjectsRequests } from '../requests/ProjectsRequests';
import { API_BASE_URL } from '../core/constants';

export type { ApiProject } from '../requests/ProjectsRequests';

export interface Api {
  projects: ProjectsRequests;
  // add more resource classes as needed
}

function buildApi(request: APIRequestContext): Api {
  return { projects: new ProjectsRequests(request) };
}

type ApiFixtures = { api: Api };

export const test = base.extend<ApiFixtures>({
  api: async ({ request }, use) => {
    await use(buildApi(request));
  },
});
```

**`fixtures/scenarios.ts`** — API-driven setup/teardown fixtures. Cross-file dependencies (e.g. `api`) are declared as a `Deps` type — `mergeTests` resolves them at runtime:

```ts
import { test as base } from '@playwright/test';
import type { Api, ApiProject } from './api';

type ScenarioFixtures = { testItem: ApiProject };
type ScenarioDeps = { api: Api };

export const test = base.extend<ScenarioFixtures & ScenarioDeps>({
  testItem: async ({ api }, use) => {
    const item = await api.projects.create({ name: `E2E ${Date.now()}`, /* ... */ });
    await use(item);
    // Teardown: delete child resources first, then the parent. 404-tolerant.
    await api.projects.delete(item.id).catch(() => {});
  },
});
```

**`fixtures/index.ts`** — merge everything here; specs import only from this file:

```ts
import { mergeTests, expect } from '@playwright/test';
import { test as pageFixtures } from './pages';
import { test as authFixtures } from './auth';
import { test as apiFixtures } from './api';
import { test as scenarioFixtures } from './scenarios';

export const test = mergeTests(pageFixtures, authFixtures, apiFixtures, scenarioFixtures);
export { expect };
```

### `requests/` — one class per API resource

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

Add one class per resource (e.g. `TasksRequests`, `TeamRequests`) following the same pattern. The `api` fixture in `fixtures/api.ts` composes them.

### `tests/auth.setup.ts`

Runs once before all tests, logs in, and saves browser state. Use `fileURLToPath` to get an absolute path — this is required in ESM and prevents path resolution failures when tests are run from a different working directory (e.g. the project root):

```ts
import { test as setup } from '../fixtures';
import { fileURLToPath } from 'url';
import path from 'path';

const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);
const authFile = path.join(__dirname, '../playwright/.auth/user.json');

setup('authenticate', async ({ loginPage, credentials }) => {
  await loginPage.goto();
  await loginPage.login(credentials.email, credentials.password);
  await loginPage.page.waitForURL('/');
  await loginPage.page.context().storageState({ path: authFile });
});
```

Adapt `waitForURL` to the app's actual post-login URL.

## Step 4 — Configure playwright.config.ts

Use **absolute paths** for both `dotenv` and `storageState` — paths resolved relative to `__dirname` are stable regardless of which directory `npm run test:e2e` is run from:

```ts
import { defineConfig, devices } from '@playwright/test';
import * as dotenv from 'dotenv';
import { fileURLToPath } from 'url';
import path from 'path';

const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);
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
    baseURL: 'http://localhost:8080',
    trace: 'on-first-retry',
  },
  projects: [
    {
      name: 'setup',
      testMatch: /.*\.setup\.ts/,
    },
    {
      name: 'chromium',
      use: {
        ...devices['Desktop Chrome'],
        storageState: path.resolve(__dirname, 'playwright/.auth/user.json'),
      },
      dependencies: ['setup'],
    },
  ],
});
```

Ask the user for the `baseURL` if it wasn't provided. Setting it here means all `page.goto()` calls and `toHaveURL()` assertions use relative paths — never hardcode the full origin.

Add a test:e2e script to the root `package.json` so tests can be run from the project root:

```json
{
  "scripts": {
    "test:e2e": "playwright test --config e2e/playwright.config.ts"
  }
}
```

## Step 5 — Set up tsconfig.json

**`e2e/tsconfig.json`** — covers the whole e2e project:

```json
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "CommonJS",
    "moduleResolution": "node",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "baseUrl": "."
  },
  "include": ["**/*.ts"],
  "exclude": ["node_modules"]
}
```

**`e2e/tests/tsconfig.json`** — picked up automatically by Playwright when running tests:

```json
{
  "extends": "../tsconfig.json",
  "compilerOptions": { "strict": true },
  "include": ["**/*.ts", "../fixtures/**/*.ts", "../pages/**/*.ts", "../requests/**/*.ts"]
}
```

## Step 6 — Create .mcp.json

The Playwright MCP server must use the **same local `playwright` package** as the project — otherwise the test runner throws "did not expect test() to be called here".

Create `e2e/.mcp.json`:

```json
{
  "mcpServers": {
    "playwright-test": {
      "command": "./node_modules/.bin/playwright",
      "args": ["run-test-mcp-server"]
    }
  }
}
```

Using `./node_modules/.bin/playwright` guarantees the MCP server runs with the exact local copy.

## Step 7 — Create .env and update .gitignore

Create `e2e/.env`:

```
TEST_EMAIL=user@example.com
TEST_PASSWORD=secret
```

Add to the root `.gitignore`:

```
/e2e/.env
/e2e/test-results/
/e2e/playwright-report/
/e2e/playwright/.auth/
/e2e/playwright/.cache/
```

`playwright/.auth/` contains session cookies — **never commit this directory**.

## Step 8 — Verify the setup

```bash
npx tsc -p e2e/tsconfig.json --noEmit   # type-check
npx playwright test --config e2e/playwright.config.ts --list  # confirm test discovery
```

No tests yet is fine — `--list` should exit cleanly. Fix any errors before finishing.

## What NOT to do

- Don't generate any test files during project init — the structure must be empty and ready for the user to add tests via `/playwright-test-generator`.
- Don't install or configure ESLint — this skill only scaffolds the project structure.
- Don't use `dotenv.config()` without an absolute path — it resolves from CWD, which breaks when `npm run test:e2e` is run from the project root instead of `e2e/`.
- Don't use a relative `storageState` path in `playwright.config.ts` for the same reason.
- Don't use `__dirname` directly in ESM files — use `fileURLToPath(import.meta.url)` first.
- Don't put all fixtures in one file — split by concern and merge with `mergeTests`.
- Don't put `pages/`, `fixtures/`, or `requests/` inside `tests/` — keep them at the `e2e/` root.
- Don't hardcode absolute URLs in `goto()` calls — `baseURL` in the config handles the origin.
- Don't import `test` or `expect` directly from `@playwright/test` in spec files — always import from `'../fixtures'`.

## Summary for the user

After setup, tell the user:

1. The project is ready. `pages/` is where Page Object classes go, `fixtures/` is where they get composed (split by concern and merged).
2. All spec files import from `'../fixtures'`, never from `@playwright/test` directly.
3. `baseURL` is set — use relative paths everywhere (`page.goto('/')` not `page.goto('https://...')`).
4. For API setup/teardown in tests, add request classes to `requests/`, expose them via `fixtures/api.ts`, and create scenario fixtures in `fixtures/scenarios.ts`.
5. Suggest running `/playwright-test-planner` next to explore the app and generate a test plan, then `/playwright-test-generator` to implement it.
