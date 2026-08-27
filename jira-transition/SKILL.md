---
name: jira-transition
description: Move a Jira ticket through its lifecycle (start work → In Progress, PR opened → Code Review with required QA Notes + PII fields, plus Blocked/Stop Work/Close) using self-healing per-project transition configs shared with jira-create-ticket
---

# /jira-transition

Progress a Jira ticket's status as work happens. Transition targets and the screen
fields each one requires are looked up per project from shared config, so the same
skill works across any Fandango project without hardcoding workflow ids.

## When to Use

Run these **skill-driven, no confirmation** — perform them automatically at the right
moment:
- **Starting work** on a ticket (branch cut, or user says "start work on X") → move to
  In Progress.
- **Opening a PR** for a ticket → move to Code Review, filling its required screen
  fields.

Also handles Blocked / Stop Work / Close on request.

## Config

Reads the **shared** per-project config owned by the create skill:
`~/.claude/skills/jira-create-ticket/configs/<KEY>.json` → its `transitions` section.
`cloudId` comes from `~/.claude/skills/jira-create-ticket/configs/_projects.json`
(project entry, else `_defaults.cloudId`).

Each named transition config carries:
- `toName` — the target status name to **match on** (`to.name` from the live
  transition list). This is the source of truth, not any id.
- `idHint` — a previously-seen id, for reference only. **Never** pass it blind.
- `fetchNote` — extra fetch flags needed to surface this transition.
- `requiredFields` — fields the transition screen requires (even if metadata reports
  them optional), each with `fieldType`, optional `default`, `allowedValues`, `note`.
- `trigger` — when this transition fires.

## Golden rule

**Transition IDs are workflow- and status-specific — never hardcode them.** Always call
`getTransitionsForJiraIssue` and match by the transition's target status name
(`to.name`), then use that transition's `id`.

## Workflow

### Step 1: Resolve project + load transition config

Derive the project key from the ticket key (`TOMATORIAL-297` → `TOMATORIAL`). Read
`configs/<KEY>.json`. If no `transitions` section exists for this project, discover
live (Step 3 handles the write-back).

### Step 2: Fetch available transitions

```js
getTransitionsForJiraIssue({
  cloudId,
  issueIdOrKey,
  expand: "transitions.fields",
  includeUnavailableTransitions: true,   // Code Review often hides from the short list
})
```

Match the transition whose `to.name` equals the config's `toName` for the target
lifecycle step. If the ticket is already in that status (Jira's GitHub integration may
auto-move it on branch/PR link), do nothing.

### Step 3: Build required fields

For the matched transition, assemble `fields` from the config's `requiredFields`:
- **ADF fields** (`fieldType: "adf"`, e.g. QA Notes) — build a real ADF doc, never a
  plain string. See `jira-create-ticket/templates/qa-notes.md` for the QA Notes
  heuristic and ADF wrappers.
- **select / multiselect** — use the config `default` if present, else the appropriate
  `allowedValues` entry, else surface options via `AskUserQuestion`.
- If the live transition screen reveals a **required field not in the config**, add it:
  ask the user (or apply an obvious default), then append it to the config's
  `requiredFields` with `learnedFrom`/`learnedOn` so it's never hit blind again.

### Step 4: Transition

```js
transitionJiraIssue({
  cloudId,
  issueIdOrKey,
  transition: { id: "<matched id from Step 2>" },
  fields: { /* from Step 3 */ },
})
```

**On a 400 for a missing/invalid screen field:** read the error, add or fix that field,
retry, and write the learning back to the config.

## TOMATORIAL lifecycle (seeded)

- **Start work → In Progress** (`toName: "In Progress"`). No required fields.
- **PR opened → Code Review** (`toName: "Ready for Code Review"`). Requires:
  - `customfield_14226` **QA Notes** — ADF doc. Numbered test steps for user-facing
    changes, or "Covered by regression." otherwise (see qa-notes template).
  - `customfield_13504` **Covered Information Data Inventory** (PII) — default
    "No Impact" (`[{ "id": "14310" }]`); pick matching option(s) if PII is involved.
- **Blocked** (`toName: "Blocked"`) — stuck on external dependency.
- **Stop Work** (`toName: "To Do"`).
- **Close** (`toName: "Closed"`) — requires a `resolution` (Done = `{ "id": "10000" }`).

## Guardrails

- Never hardcode a transition id — match by `to.name` every time; `idHint` is a
  reference, not an input.
- Never write a plain string into an ADF field (QA Notes 400s: "must be an Atlassian
  Document").
- Never skip a `requiredField` the config lists — the transition 400s even when Jira's
  field metadata reports it optional.
- For Code Review, always fetch with `includeUnavailableTransitions: true` and
  `expand: transitions.fields`.
- Every self-healing write to a project config must include `learnedFrom` + `learnedOn`.
- If the ticket is already in the target status, do nothing — don't error.
