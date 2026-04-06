---
name: assess-risk
description: >
  Assess risk and prioritise test cases using ISTQB risk-based testing. Scores each area by
  likelihood of failure and business impact to produce a ranked execution order. Use after
  generate-test-cases or analyze-story, or whenever the user needs to decide what to test first,
  triage a backlog of test scenarios, or justify test coverage decisions to stakeholders. Trigger
  on phrases like "prioritise tests", "what should I test first", "risk assessment", "rank test cases",
  "risk-based testing", or "which tests are most important".
---

# Assess Risk

Apply risk-based testing to prioritise what to test first. Based on ISTQB risk analysis: **Risk = Likelihood × Impact**.

## Input

If not already provided, ask for:
- The list of test cases or feature areas to assess
- Any context about the domain, recent changes, or known fragile areas

**If called directly without prior test cases**, that's fine — score at the feature-area level instead.
Use the story's AC or the user's description of the feature as the unit of scoring. Each AC criterion
or functional area becomes a row in the risk matrix. The same Likelihood × Impact formula applies.

---

## Step 1 — Define Risk Areas

Group test cases into risk areas (feature areas or AC groups) if there are many. Risk is assessed at the area level first, then individual test cases are ranked within each area.

Example risk areas:
- Authentication flow
- Payment processing
- Search and filter
- User profile management

---

## Step 2 — Score Likelihood (1–5)

How likely is this area to have a defect?

| Score | Description                                                      |
|-------|------------------------------------------------------------------|
| 5     | New or heavily changed code; complex logic; unclear requirements |
| 4     | Moderate changes; some known complexity; limited test history    |
| 3     | Some changes; reasonably well-understood; some past defects      |
| 2     | Minor changes; stable area; few past defects                     |
| 1     | No changes; very stable; extensively tested historically         |

**Factors that increase likelihood**:
- Ambiguous or missing acceptance criteria
- Complex business rules (decision tables, many branches)
- Integration with external systems
- Concurrency / race conditions
- New team member implemented it
- Short development time / pressure

---

## Step 3 — Score Impact (1–5)

What is the consequence if this fails in production?

| Score | Description                                                                  |
|-------|------------------------------------------------------------------------------|
| 5     | Financial loss, data corruption, legal/regulatory violation, complete outage |
| 4     | Major user-facing failure, loss of core business function                    |
| 3     | Degraded experience, workaround exists, moderate user impact                 |
| 2     | Minor UX issue, cosmetic, low user visibility                                |
| 1     | Internal only, no user impact, easily recoverable                            |

**Factors that increase impact**:
- Payments, personal data, or authentication involved
- High-traffic or critical user journey
- Regulatory requirements (GDPR, PCI-DSS, accessibility laws)
- No fallback or recovery mechanism
- Data loss or corruption possible

---

## Step 4 — Calculate Risk Score and Classify

Risk Score = Likelihood × Impact

| Score | Level       | Action                                     |
|-------|-------------|--------------------------------------------|
| 17–25 | 🔴 Critical | Test first; block release if failing       |
| 10–16 | 🟠 High     | Must test before release                   |
| 5–9   | 🟡 Medium   | Test if time allows; include in regression |
| 1–4   | 🟢 Low      | Exploratory / low-priority regression      |

---

## Step 5 — Output Risk Matrix

```markdown
## Risk Assessment

| Area / Test Case                        | Likelihood | Impact | Risk Score | Level       | Recommendation |
|-----------------------------------------|------------|--------|------------|-------------|----------------|
| Payment flow — checkout to confirmation | 4          | 5      | 20         | 🔴 Critical | Test first     |
| Login with invalid credentials          | 3          | 5      | 15         | 🟠 High     | Must test      |
| Profile photo upload                    | 2          | 2      | 4          | 🟢 Low      | Exploratory    |
```

Sort by Risk Score descending.

---

## Step 6 — Recommended Test Execution Order

Based on the matrix, produce a recommended execution order:

```markdown
## Recommended Execution Order

### Phase 1 — Critical (test before any release)
1. Payment flow — checkout to confirmation
2. Auth token expiry during active session

### Phase 2 — High (required for release sign-off)
3. Login with invalid credentials
4. Search filter with special characters

### Phase 3 — Medium (if time allows)
5. Profile update — name and email fields
6. Pagination on large data sets

### Phase 4 — Low (regression / exploratory)
7. Profile photo upload
```

---

## Step 7 — Save output to file

After presenting the risk matrix and execution order, save them as a markdown file.

Save to: `qa-output/risk-assessments/<topic>-risk-<YYYY-MM-DD>.md`

- Derive `<topic>` from the ticket key, story title, or feature name (e.g. `PROJ-123`, `checkout-flow`)
- Use today's date for `<YYYY-MM-DD>`
- Create the `qa-output/risk-assessments/` directory if it does not exist
- The file content is the full risk matrix and recommended execution order exactly as presented to the user

Tell the user where the file was saved: `"Risk assessment saved to qa-output/risk-assessments/<filename>.md"`

---

## After Assessment

Ask the user:
1. Does this prioritisation match your understanding of the system risk?
2. Are there external factors (upcoming release, regulatory deadline, recent prod incident) that should shift any priorities?

Once the user starts executing the prioritised tests, remind them:

> "When a test fails, use `/report-bug` to structure the defect and log it in Jira — with the risk level and story context already attached."

If the user came here directly without running the earlier steps, suggest:

> "Want to go deeper? Start with `/analyze-story` to check story quality, then `/generate-test-cases` to derive the full test set before prioritising."
