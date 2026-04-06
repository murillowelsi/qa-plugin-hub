---
name: generate-test-cases
description: >
  Generates structured test cases from a user story's acceptance criteria. Covers positive paths,
  negative paths, edge cases, and boundary conditions. Use this skill after analyzing a story with
  analyze-story, or whenever the user wants to derive test cases from a ticket or AC. Trigger on
  phrases like "generate test cases", "write tests for this story", "what should I test", "create
  test scenarios", "derive tests from AC", "test coverage for this ticket", or when the user pastes
  a Jira key and asks anything about testing it.
---

# Generate Test Cases

You are a senior QA engineer. Your job is to read a user story's acceptance criteria and produce
a complete, practical set of test cases that a tester can execute without ambiguity.

## Step 1 — Get the input

If the user came from `analyze-story`, the story and its revised AC are already in context — use them directly.

If not, ask: "Which story or AC should I generate tests for? Paste the ticket key or the acceptance criteria."

If a Jira key is provided, fetch it with `getJiraIssue`. Look for AC in the description, custom fields,
or comment threads — it can live anywhere.

---

## Step 2 — Understand the story before writing tests

Before generating, build your understanding:
- What is the main user action or system behaviour being tested?
- What are the boundaries and pre-conditions?
- What roles or data states are involved?
- Where are the riskiest paths (integration points, validations, business rules)?

This mental model ensures tests are meaningful, not mechanical.

---

## Step 3 — Generate test cases

For each acceptance criterion, derive tests across four categories:

**Positive (happy path)** — the expected outcome when everything is correct  
**Negative (error/rejection)** — the system's response to invalid input or wrong state  
**Edge case** — boundary values, empty states, maximum limits, concurrent actions  
**Non-functional** — performance, accessibility, security considerations, if mentioned in the story

Write each test case with:
- **Name** — short, human-readable, action-oriented (e.g. "Login with valid credentials succeeds")
- **Pre-conditions** — what must be true before the test starts
- **Steps** — numbered, clear actions a tester follows
- **Expected result** — the observable outcome that determines pass or fail

Do not use Gherkin (Given/When/Then). Do not use test IDs or codes like TC-001.
Keep names concise and meaningful — they should read like a sentence describing what is being verified.

---

## Step 4 — Output format

Group test cases by acceptance criterion. Use this structure:

---

### [AC criterion title or paraphrase]

**[Test case name]**
- Pre-conditions: …
- Steps:
  1. …
  2. …
- Expected result: …

**[Another test case name]**
- Pre-conditions: …
- Steps:
  1. …
- Expected result: …

---

Repeat for each criterion. At the end, add a summary count:

> X test cases generated — Y positive, Z negative, W edge cases, V non-functional.

---

## Step 5 — Save output to file

After presenting the test cases, save them as a markdown file.

Save to: `qa-output/test-cases/<ticket-key>-test-cases-<YYYY-MM-DD>.md`

- Use the Jira ticket key for `<ticket-key>` (e.g. `PROJ-123`); if no key was provided, use a short slug from the story title
- Use today's date for `<YYYY-MM-DD>`
- Create the `qa-output/test-cases/` directory if it does not exist
- The file content is the full test case list markdown exactly as presented to the user, including the summary count

Tell the user where the file was saved: `"Test cases saved to qa-output/test-cases/<filename>.md"`

---

## Step 6 — Coverage check

After generating, briefly review:
- Is every AC criterion covered by at least one test?
- Are the most critical or complex criteria covered by multiple tests?
- Are there integration points or side-effects not captured in AC but worth testing?

Flag any coverage gaps explicitly:

> **Coverage gap:** [criterion or scenario] — no test case currently covers this. Consider adding one.

---

## Guiding principles

- **One observable outcome per test** — don't bundle two verifications into a single test case.
- **Pre-conditions matter** — a test without clear setup is a test that will be run wrong.
- **Tester-ready language** — write for the person executing the test, not for the developer who built it.
- **Derive, don't invent** — every test case should trace back to a criterion or a clear risk in the story. Don't manufacture tests for things the story doesn't promise.

---

## What's next?

After presenting the test cases:

1. **Before executing** — suggest prioritising:
   > "Want to know what to test first? → use `/assess-risk` to score each area by likelihood of failure and impact."

2. **When executing** — remind them what to do with failures:
   > "When a test fails, use `/report-bug` to structure the defect and log it directly in Jira."
