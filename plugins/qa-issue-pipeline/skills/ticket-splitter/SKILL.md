---
name: ticket-splitter
description: >
  Analyzes a Jira ticket that is too large, contains multiple independent deliverables,
  or shows signs of technical-layer fragmentation, and proposes a concrete split into
  smaller, independently releasable stories with titles, rationale, and suggested ACs.
  Applies the team's ticket structuring rules: 1 ticket = 1 deliverable, technical layers
  belong as subtasks (not standalone tickets), and each split story must be independently
  testable and releasable.
  Use this skill whenever the user asks "how should I split this?", "this story is too big",
  "can you break this down?", "split this ticket", "decompose this story", "help me split this issue",
  or when issue-analyzer flags structural issues under dimension #8 (Work Item Structure & Estimation).
  Always use this skill instead of manually guessing how to split a ticket.
allowed-tools: Read, Write, mcp__*__getJiraIssue, mcp__*__getConfluencePage, mcp__*__createJiraIssue, mcp__*__addCommentToJiraIssue
---

# Ticket Splitter

You are running the **ticket-splitter** skill. Analyze a Jira ticket and propose a concrete, actionable split into smaller stories that each deliver independent value. Do all the work directly — no sub-agents.

## Step 1 — Get the ticket key

If the user has not provided a Jira ticket key, ask for one now.

## Step 2 — Gather context

Fetch the full ticket using the Jira MCP tool. Also fetch:
- **Subtasks**: to understand what work has already been decomposed
- **Parent ticket** (if any): to understand whether this ticket itself might already be a subtask that should stay that way
- **Linked issues**: for scope context
- **Confluence pages** referenced in the description: for additional background

Then check for an existing analysis report at `qa-output/issue-pipeline/<KEY>/01-analysis.md`. If it exists, read it — the findings (especially dimension #8) will inform your split rationale.

## Step 3 — Assess

Before proposing a split, decide whether a split is actually warranted. A ticket should be split only when:

- It contains **multiple independent deliverables** — each of which can be tested and potentially released on its own
- It is **too broad for a single sprint** and the work can be meaningfully separated
- It mixes **user-facing value** with infrastructure or technical work that could be sequenced separately

A ticket should **not** be split just because it is complex. Complexity is handled with subtasks, not with separate stories. If the ticket is large but represents a single indivisible deliverable, say so clearly and explain why.

If no split is warranted, stop here and tell the user:
> "This ticket is large but delivers a single indivisible outcome. Breaking it up would create artificial dependencies with no independent release value. I'd suggest adding subtasks instead to break down the implementation work."

## Step 4 — Identify split boundaries

Apply the following rules to determine how to divide the work:

**What becomes a new independent story:**
- A distinct user-facing capability that can be shipped and validated on its own
- A phase of delivery with its own testable acceptance criteria (e.g., a read-only view before a full edit workflow)
- Infrastructure work that unblocks the team but has its own definition of done independent of the feature work

**What stays as a subtask (not a separate story):**
- Technical implementation layers: API endpoints, UI components, database migrations, unit tests — these are steps toward the same deliverable, not separate deliverables
- Work that can only be validated in combination with other parts of the same ticket
- Anything that cannot be independently released or demonstrated to a stakeholder

**Anti-patterns to avoid in your proposal:**
- Do not split by technical layer (FE / BE / API / tests as separate stories) — those are subtasks
- Do not propose a split that creates hard sequential dependencies with no independent value at each step
- Do not create extra tickets just to distribute story points across the team

## Step 5 — Write the split proposal

For each proposed story, produce:

```markdown
### Story N — [Proposed title]

**Independent value**: [One sentence — what does a user or stakeholder get from this story alone, without the others?]
**Can be released without the others?**: Yes / No (if No, explain why it's still a valid split)
**Suggested story points**: [S / M / L / XL or a number if the team uses numeric points]

**Suggested ACs**:
- [AC 1 — concrete and testable]
- [AC 2]
- [AC 3]

**Implementation hints** (subtasks):
- [ ] [Step 1 — e.g., API endpoint for X]
- [ ] [Step 2 — e.g., UI component for Y]
- [ ] [Step 3 — e.g., integration tests]
```

Then add a section on what happens to the original ticket:

```markdown
## What to do with [TICKET-KEY]

[Choose one:]
- **Repurpose as Story 1** and create the remaining stories as new tickets
- **Close and replace** — archive the original and create all stories as new tickets
- **Keep as epic / parent** — downgrade the original to a tracking ticket and link the new stories under it
```

## Step 6 — Save and present for review

Save to `qa-output/issue-pipeline/<KEY>/split-proposal.md` using this structure:

```markdown
# Split Proposal — [TICKET-KEY]: [Original Title]

## Why split this ticket
[2-3 sentences explaining the split rationale based on the analysis]

## Proposed stories

### Story 1 — [Title]
...

### Story 2 — [Title]
...

## What to do with [TICKET-KEY]
...

---
_Generated by QA Ticket Splitter on [date]_
```

Display the full proposal to the user, then ask:

> "Does this split make sense? You can:
> - Reply **yes** to create the new stories in Jira
> - Tell me what to adjust (e.g., 'merge story 1 and 2', 'add an AC for X', 'story 3 should also cover Y')
> - Reply **no** to discard"

If the user requests changes, revise and re-present before asking for approval again.

**Do not create anything in Jira until the user explicitly approves.**

## Step 7 — Create tickets in Jira (only after approval)

Once the user approves, for each new story:

1. Create a new Jira issue using the MCP tool:
   - **Issue type**: User Story (or Task, following the team's classification rules)
   - **Summary**: the proposed title
   - **Description**: build a proper user story description from the proposed value and ACs
   - **Link to original**: link the new story to `[TICKET-KEY]` as "is split from" or "relates to"

2. If the original ticket should be repurposed as Story 1, update its summary and description via the MCP tool.

3. Post a comment on the original ticket:

```
✂️ *Ticket split by QA Ticket Splitter.*

This story was split into [N] smaller, independently deliverable stories:
- [PROJ-XXX]: [Story 1 title]
- [PROJ-YYY]: [Story 2 title]

*Reason:* [One sentence on why the split was made]

Next step: run */issue-analyzer* on each new story to verify quality before sprint planning.

_Applied by QA Issue Pipeline on [date]_
```

## Step 8 — Final summary

Display:

```
╔══════════════════════════════════════════════════════════╗
║   Ticket Splitter — [KEY]: [Original Title]              ║
╠═══════════════════╦══════════════════════════════════════╣
║ Original ticket   ║ [Repurposed / Closed / Kept as epic] ║
║ New stories       ║ [N] created                          ║
║ Story keys        ║ [PROJ-XXX], [PROJ-YYY], ...          ║
║ Jira comment      ║ ✅ Posted to [KEY]                   ║
╚═══════════════════╩══════════════════════════════════════╝

Output saved to: qa-output/issue-pipeline/[KEY]/split-proposal.md

Run /issue-analyzer on each new story to validate quality before pulling into the sprint.
```

## Guiding principles

- A split is only valid if each resulting story can be demonstrated to a stakeholder independently
- Technical layers (FE/BE/API/tests) are always subtasks — never use them as the split boundary
- Preserve the original story's intent — don't change what the team is building, just how the work is sequenced
- Never create tickets in Jira without explicit user approval
- If the Jira creation fails for any story, save the content to disk and tell the user to create it manually
