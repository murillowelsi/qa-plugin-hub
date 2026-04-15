---
name: issue-refiner
description: >
  Rewrites a blocked Jira issue to meet the Definition of Ready.
  Reads the DoR block verdict and analysis findings, generates a refined title,
  structured description, and complete Given/When/Then acceptance criteria that
  address every failing rule and CRITICAL/HIGH finding. Shows the rewrite for
  review before applying any changes to Jira.
  Use this skill when an issue has been blocked by the DoR gate and the user
  wants to fix it, asks "refine this issue", "refine this story", "fix the blocked issue", "fix the blocked story", "rewrite
  the ACs", "update the ticket", or "clean up this story".
allowed-tools: Read, Write, mcp__*__getJiraIssue, mcp__*__editJiraIssue, mcp__*__addCommentToJiraIssue, mcp__*__getJiraProjectIssueTypesMetadata
---

# Issue Refiner

You are running the **issue-refiner** skill. Take a blocked Jira issue and rewrite it to a production-quality standard, present the rewrite for user review, and only apply changes to Jira after explicit approval. Do all the work directly — no sub-agents.

## Step 1 — Get the ticket key

If the user has not provided a Jira ticket key, ask for one now.

## Step 2 — Load prior outputs

Check for:
- `qa-output/issue-pipeline/<KEY>/01-analysis.md` (or `1-story-analysis.md`)
- `qa-output/issue-pipeline/<KEY>/dor-verdict.json`

If neither exists:
> "This story hasn't been through the pipeline yet. Run `/issue-pipeline [KEY]` first so I have the analysis and DoR verdict to work from."

Read both files now.

## Step 3 — Fetch the current Jira ticket

Fetch the full ticket using the Jira MCP tool. Extract:
- Current title (summary)
- Current description and acceptance criteria
- Story points (if set)
- Any linked issues or attachments

## Step 3.5 — Issue Type Guard

Check the `issuetype` field from the fetched ticket. Only **User Story**, **Bug**, and **Task** are permitted.

If the type is anything else, do not rewrite the ticket. Instead, read the title and description to understand its nature, then produce a reclassification recommendation:

**Reclassification logic:**
- **Spike** → almost always should be a **Task** (investigation work is internal, non-user-facing, pre-production)
- **Sub-task** → flag the parent story for refinement instead; sub-tasks are not independently refined
- **Epic** → should be decomposed into **User Stories** first, then each story refined individually
- **Any other type** → recommend the closest match (Story, Bug, or Task) based on the content

For **Sub-task** and **Epic**, these cannot be directly reclassified without broader implications. Respond with the type violation message (see below) and stop — do not offer to fix in Jira.

For **Spike** and other single-reclassifiable types, present the violation and offer to fix it:

> "**Issue Type Violation — [KEY]: [Title]**
>
> Current type: **[Type]** — this is not a permitted issue type.
> Only **User Story**, **Bug**, and **Task** are allowed in this pipeline.
>
> **Recommended reclassification: [Suggested type]**
> Reason: [1–2 sentences explaining why, based on the ticket's actual content]
>
> Would you like me to update the issue type to **[Suggested type]** in Jira now and continue with the refinement?"

If the user confirms (yes / go ahead / do it / etc.):
1. Use `getJiraProjectIssueTypesMetadata` to find the correct issue type ID for the suggested type in this project.
2. Use `editJiraIssue` to update the `issuetype` field to the found ID.
3. Confirm the update: "Issue type updated to [Suggested type]. Continuing with refinement..."
4. Proceed to Step 4 as normal.

If the user declines or does not respond affirmatively, stop here and do not rewrite, save files, or update Jira.

## Step 4 — Rewrite the story

Using the analysis findings and DoR verdict as your guide, produce:

### Refined Title
- Pattern: `As a [user], I want to [goal]` or a clear action-oriented title
- No technical scope tags (`[BE]`, `[FE]`, `[API]`, etc.)
- Specific enough that any team member understands scope without reading the description

### Refined Description
Use this structure (Jira wiki markup):

```
*As a* [user role],
*I want to* [goal / capability],
*So that* [business value / outcome].

---

*Background / Context*
[1-3 sentences of essential context. Omit if not needed.]

*Out of scope*
- [At least 1 explicit exclusion to prevent scope creep]
```

### Refined Acceptance Criteria
Rewrite every existing AC into Given/When/Then format. Then add the missing scenarios from the analysis:
- At least 1 negative/unhappy path per main AC
- Edge cases for timing, boundaries, and error states identified in the findings
- Notification/email content where relevant
- Failure handling ACs

Format (Jira wiki markup):

*AC #N — [Short label]*
```gherkin
Given [precondition]
When [action]
Then [expected outcome]
```

### Notes / Open Questions (optional)
If the analysis identified missing designs, dependencies, or open questions, list them so the PO knows what to address.

**Quality bar**: every rewritten story must address all failing DoR rules and have zero CRITICAL findings if re-analyzed.

## Step 5 — Save and present the refined story for review

Save to `qa-output/issue-pipeline/<KEY>/refined-story.md`:

```markdown
# Refined Story — [TICKET-KEY]

## Title
[Refined title]

## Description
[Full description in Jira wiki markup]

## Acceptance Criteria
[All ACs in Given/When/Then, numbered]

## Notes / Open Questions
[Optional — omit if empty]

---
_Refined by QA Issue Refiner on [date]. Addresses [N] CRITICAL and [N] HIGH findings._
```

Then display the full refined content to the user — title, description, and all ACs — so they can read and evaluate it. Follow with a summary of what was changed:

```
📝 Refined story saved to qa-output/issue-pipeline/[KEY]/refined-story.md

Changes made:
- Title: "[Old title]" → "[New title]"
- Description: rewritten with user story format + out-of-scope section
- ACs: X → Y (Z new added — list them briefly)
- Findings addressed: [list CRITICAL and HIGH findings fixed]
```

**Stop here and ask the user for feedback before touching Jira:**

> "Does this look good? You can:
> - Reply **yes** to apply these changes to the Jira ticket
> - Tell me what to change (e.g. 'reword AC #3', 'remove the out-of-scope section', 'add an AC for X')
> - Reply **no** to discard"

If the user requests changes, revise the content accordingly, update the saved file, and present the updated version again before asking for approval once more.

Do NOT proceed to Step 6 until the user explicitly confirms with **yes** or equivalent approval.

## Step 6 — Apply updates to Jira (only after approval)

Once the user approves, update the ticket using the MCP tool:
- `summary` → refined title
- `description` → refined description + ACs combined (Jira wiki markup)

Then post a comment:
```
✏️ *Story refined by QA Issue Refiner.*

*Issues addressed:*
- [Each CRITICAL and HIGH finding that was fixed]

*Changes applied:*
- ✅ Title updated
- ✅ Description rewritten (user story format + out-of-scope section)
- ✅ Acceptance criteria rewritten in Given/When/Then format
- ✅ [X] new edge case / negative path ACs added

Next step: re-run */issue-pipeline [KEY]* to validate the refined story.

_Applied by QA Issue Pipeline on [date]_
```

## Step 7 — Final summary

Display to the user:

```
╔══════════════════════════════════════════════════════════╗
║   Story Refiner — [KEY]: [Old Title] → [New Title]       ║
╠═══════════════════════╦══════════════════════════════════╣
║ What changed          ║ Detail                           ║
╠═══════════════════════╬══════════════════════════════════╣
║ Title                 ║ Updated                          ║
║ Description           ║ Rewritten (user story format)    ║
║ Acceptance Criteria   ║ X ACs → Y ACs (Z new added)      ║
║ CRITICAL findings     ║ X addressed                      ║
║ HIGH findings         ║ X addressed                      ║
║ Jira updated          ║ ✅ Summary + description patched  ║
║ Comment posted        ║ ✅ Refinement log added           ║
╚═══════════════════════╩══════════════════════════════════╝

Output saved to: qa-output/issue-pipeline/[KEY]/refined-story.md

Re-run the pipeline to validate: /issue-pipeline [KEY]
```

## Guiding principles

- Rewrite only what the analysis and DoR verdict identify as wrong — preserve the original story's intent
- Never use placeholders or vague language — the refined story must be immediately usable by a developer
- **Never update Jira without explicit user approval** — always stop at Step 5 and wait for confirmation
- If the user requests adjustments, revise the content and present it again before asking for approval
- If the Jira update fails, save to disk and tell the user to copy-paste manually
