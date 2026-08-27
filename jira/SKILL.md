---
name: jira
description: Use when creating or progressing JIRA issues in TOMATORIAL — issue structure, testable acceptance criteria, and status transitions (start work → In Progress, PR opened → Code Review with QA Notes).
model: claude-haiku-4-5-20251001
---

# JIRA (TOMATORIAL)

Covers the full ticket lifecycle: **creating** issues and **progressing** them through
status as work happens. Two parts:

1. **Creating issues** — structure, acceptance criteria, defaults.
2. **Lifecycle transitions** — move a ticket to *In Progress* when work starts, and to
   *Code Review* when a PR is opened (setting the required QA Notes + PII fields).

## Project Constants

Reuse these instead of rediscovering them each time:

| Thing | Value |
|-------|-------|
| cloudId | `88b0b656-d57e-42ea-87c1-2808cc792710` |
| projectKey | `TOMATORIAL` |
| Default issue type | `Story` |
| QA Notes field | `customfield_14226` (ADF / rich-text — **not** a plain string) |
| Covered Information Data Inventory | `customfield_13504` — "No Impact" = option id `14310` |
| Key statuses | To Do → In Progress (`10171`) → Code Review (`10549`); also Blocked, Closed |

**Transition IDs are workflow- and status-specific — never hardcode them.** Always call
`getTransitionsForJiraIssue` and match by the transition's **target status name**
(`to.name`), then use that transition's `id`.

---

# Part 1 — Creating Issues

## Overview

JIRA issues for TOMATORIAL follow a specific structure:
- **Summary:** One-line, concise problem/feature statement
- **Description:** Context and scope without external research
- **Acceptance Criteria:** Bolded header, bulleted list of testable goals (one-liners)
- **Defaults:** Always Story type, components/priority default, assignee is you, project is TOMATORIAL

## When to Use

Use this section when:
- User asks to create a JIRA issue
- No explicit issue type given → default to Story
- Parent epic unknown → prompt user, offer to mock one if needed
- Summary/description/AC need structuring

## Before Starting

Gather from user (or infer from context):
- **What problem/feature?** (required)
- **Parent epic?** (required — ask explicitly; must exist in JIRA)
- **Acceptance criteria?** (user provides or you synthesize from problem statement)

If user is vague on any, **ask once and offer a sensible default**:
- Epic: "Which epic should this be under?" If user doesn't know existing epics, offer to search JIRA first
- Criteria: "I'll suggest testable goals based on the problem"

**Critical:** Parent epic MUST exist and be specified correctly. If the epic doesn't exist, JIRA will throw an error or auto-link to a different epic. Always verify the epic key with the user before creating the issue.

## Writing the Issue

**CRITICAL: Every issue MUST include Acceptance Criteria. No exceptions.**

### Summary (One-Liner)
- Starts with verb or noun phrase
- Max ~50 chars
- Specific, not vague
- No "Fix" or "Add" prefix (action is clear from context)

**Examples:**
- ✅ "Server-side vote limiting for rt-polls"
- ✅ "Add Redis caching to Hub post queries"
- ❌ "Fix rt-polls" (too vague)
- ❌ "Add a feature" (no specifics)

### Description
Write in this order:
1. **Problem:** What's broken or missing? (1-2 sentences)
2. **Scope:** What's in scope, what's not? (1-2 bullets if needed)
3. **Context:** Why does this matter? (1 sentence max)
4. **Acceptance Criteria:** Bolded header + bulleted testable goals (REQUIRED — always include this section)

**Do NOT:**
- Research external systems
- Link to external docs
- Write implementation details (that's the developer's job)
- Make assumptions about solutions
- Skip AC or omit testable goals

**Example:**
```
rt-polls currently allows unlimited votes from a single user, enabling spam and skewing results. 
We need server-side validation to restrict votes per user per poll.

**Scope:**
- Add vote-per-user limit to backend
- Return 429 on duplicate votes

**Why:** Current client-side checks can be bypassed; backend validation is essential for data integrity.
```

### Acceptance Criteria

**Always:**
- Use a bolded `**Acceptance Criteria**` header
- Use bullet points (one per line)
- Each bullet is a testable goal, not a task

**Format:** "✅ Acceptance Criteria"
- ✅ User cannot vote more than once per poll
- ✅ Server returns 429 status on duplicate vote attempt
- ✅ Vote count reflects only valid votes (duplicates not counted)
- ✅ Admin dashboard shows accurate totals

**NOT:**
- ❌ "Implement vote limiting" (task, not testable goal)
- ❌ "Update database schema" (implementation detail)
- ❌ "Fix the vote bug" (vague)

## Creating the Issue via MCP

Before calling `createJiraIssue`, **verify the parent epic exists** by searching JIRA or asking the user.

Use `createJiraIssue` with:
- **cloudId:** `88b0b656-d57e-42ea-87c1-2808cc792710`
- **projectKey:** Always `TOMATORIAL`
- **issueTypeName:** `Story` (unless explicitly told otherwise)
- **summary:** One-liner from above
- **description:** Formatted description + AC
- **parent:** Parent epic key (e.g., `TOMATORIAL-132`)
- **additional_fields:**
  - `assignee`: User's account ID
  - `components`: Default (ask which component if unclear: rt-editorial, rt-polls)
  - `priority`: Default (ask if user specifies)

**If creation fails with "parent not found" or "must be linked to an Epic":** The epic doesn't exist or isn't in the right format. Ask user to confirm the epic key, then retry.

After creation, return the issue link and key.

---

# Part 2 — Lifecycle Transitions

These run **skill-driven, no confirmation** — perform them automatically at the right
moment. Always fetch transitions first (`getTransitionsForJiraIssue`) and match by target
status name, since IDs vary per workflow.

## Starting Work → In Progress

**Trigger:** you begin work on a ticket (e.g., user says "start work on TOMATORIAL-X",
or you cut a ticket branch).

1. `getTransitionsForJiraIssue` for the ticket.
2. Find the transition whose `to.name` is `In Progress`; use its id in `transitionJiraIssue`.
3. If the ticket is already In Progress (Jira's GitHub integration may auto-move it when a
   branch/PR links), do nothing.

## PR Opened → Code Review

**Trigger:** you open a PR for the ticket.

1. `getTransitionsForJiraIssue` with `expand: transitions.fields` and
   `includeUnavailableTransitions: true` (the Code Review transition may not appear in the
   short list).
2. Match the transition whose `to.name` is `Code Review` (named "Ready for Code Review").
3. **This transition screen requires two fields** — supply both or the transition 400s,
   even though the field metadata reports them as optional:

   - **QA Notes** (`customfield_14226`): **ADF document**, not a plain string. See heuristic below.
   - **Covered Information Data Inventory** (`customfield_13504`): PII checklist. Default to
     **"No Impact"** (`[{"id": "14310"}]`) for changes that touch no personal data. If the
     change does involve PII, pick the matching option(s) from the transition's `allowedValues`.

### QA Notes heuristic

Decide what QA can actually test:
- **User-facing / testable behavior changed** → write numbered QA test steps a tester can follow.
- **Nothing user-facing to test** (infra cleanup, dead-code removal, tooling, refactors with no
  behavior change) → set QA Notes to **"Covered by regression."**

### QA Notes must be ADF

Plain strings are rejected (`Operation value must be an Atlassian Document`). Wrap the text:

```json
{
  "customfield_14226": {
    "type": "doc",
    "version": 1,
    "content": [
      { "type": "paragraph", "content": [{ "type": "text", "text": "Covered by regression." }] }
    ]
  }
}
```

For step lists, use an `orderedList` of `listItem` → `paragraph` → `text` nodes.

### Example transition call

```
transitionJiraIssue(
  cloudId,
  issueIdOrKey,
  transition: { id: "<matched Code Review id>" },
  fields: {
    customfield_14226: <ADF doc>,
    customfield_13504: [{ id: "14310" }]   // No Impact
  }
)
```

## Other transitions

- **Blocked** — target status `Blocked`, when work is stuck on an external dependency.
- **Stop Work** — back to `To Do`.
- **Close** — target `Closed`; requires a `resolution` (e.g., `{ "id": "10000" }` = Done).

---

# Common Mistakes

**Mistake 1: Vague Acceptance Criteria**
- ❌ "Tests pass"
- ✅ "All vote-limiting tests pass with 100% coverage"

**Mistake 2: Implementation in Description**
- ❌ "Create a votes_per_user column in the database"
- ✅ "Server prevents duplicate votes from a single user"

**Mistake 3: Forgetting Parent Epic**
- Always ask. If user doesn't know, search JIRA first to find existing epics. Don't guess.

**Mistake 4: Wrong Epic Format in MCP Call**
- ❌ `"parent": "TOMATORIAL-200"` (string)
- ✅ `"parent": {"key": "TOMATORIAL-200"}` (object) — or pass the key to the `parent` param

**Mistake 5: Using Non-Existent Epic**
- ❌ Passing parent epic that doesn't exist → JIRA error or auto-links to wrong epic
- ✅ Verify epic exists before creating the issue

**Mistake 6: Too Many Details**
- Keep description short. If it spans 10+ lines, user should provide context.

**Mistake 7: Wrong issue type**
- ❌ Creating a Task by default
- ✅ Default to **Story** unless explicitly told otherwise

**Mistake 8: Plain-string QA Notes**
- ❌ `customfield_14226: "Covered by regression"` → 400 "must be an Atlassian Document"
- ✅ Wrap in an ADF doc

**Mistake 9: Omitting the PII checklist on Code Review**
- ❌ Only setting QA Notes → 400 "Covered Information Data Inventory is required"
- ✅ Also set `customfield_13504` (default "No Impact")

**Mistake 10: Hardcoding transition IDs**
- ❌ Assuming Code Review is always id 21
- ✅ Fetch transitions, match by `to.name`

## Rationalization Traps

| Excuse | Reality |
|--------|---------|
| "AC can be written after creation" | AC is part of the description. Include before creating. |
| "Description is enough; AC is redundant" | AC = testable goals. Description = context. Both required. |
| "I'll ask the user for AC during development" | AC should be clear upfront. That's the point of JIRA. |
| "This is a simple fix; doesn't need AC" | Every story needs testable goals, simple or complex. |
| "AC as prose is fine" | Bullet format = scannable, testable. Write bullets. |
| "Task is close enough to Story" | Default is Story. Use it unless told otherwise. |

## Red Flags - STOP and Clarify

**CRITICAL:**
- ❌ No "Acceptance Criteria" section in description → Add it. Every issue needs AC.
- ❌ AC section exists but is empty or missing → Write testable goals now.

**Standard:**
- User doesn't specify parent epic → Ask explicitly, search JIRA to confirm it exists
- Parent epic doesn't exist or isn't found → Stop and ask user for correct epic key
- Parent epic key is wrong format → Must be `{"key":"TOMATORIAL-XXX"}` in MCP call, not a string
- Summary is longer than one line → Compress it
- Acceptance criteria are tasks, not testable goals → Rewrite
- Issue type not specified → Default to Story
- Component unclear → Ask which area (rt-editorial, rt-polls)
- MCP creation fails with "parent" error → Verify epic exists and format is correct
- Code Review transition 400s → check both required screen fields (QA Notes ADF + PII checklist)
