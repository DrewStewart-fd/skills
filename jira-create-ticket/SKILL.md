---
name: jira-create-ticket
description: Create a Jira ticket from a free-text prompt in any Fandango project, optionally linked to another ticket (e.g. "blocks TARS-1381"), using self-healing per-project field configs and per-issue-type content templates
---

# /jira-create-ticket

Create a Jira ticket whose plumbing (required fields, parent constraints) is looked up
and cached per project+issue type, and whose content (Overview / Evidence / Steps to
Reproduce / Acceptance Criteria) is written by Claude from a fixed template so every
ticket in a project reads the same way.

Works across **any** Fandango Jira project (TOMATORIAL, TARS, CR, …). The project is
resolved from free text; per-project quirks live in `configs/`, not in this file.

## Philosophy

A ticket produced by this skill is the **baseline a human or agent picks up cold** —
not a dump of every fact gathered during investigation. Write only what's known.
Every section should make someone want to read it: concise, concrete, actionable.

- Never pad a section just to fill space. If there's no evidence, omit the Evidence
  section entirely rather than writing filler.
- **Open questions are not Acceptance Criteria.** "Confirm with @X whether Y" is not
  testable by a QA reader — it belongs in Overview as a caveat, or as its own
  follow-up ticket. Every AC bullet must be a condition someone can check off with
  no additional context.
- Steps to Reproduce should include real URLs/paths/queries wherever they exist, so
  someone can copy-paste and go instead of guessing at navigation.
- Do the research before writing the ticket, but don't transcribe the research into
  the ticket. Summarize; link to the source (e.g. the ticket that motivated this one).

## When to Use

When the user asks to create a Jira ticket from a description, investigation, or as a
follow-up/blocker to an existing ticket — e.g. "create a TOMATORIAL story for
server-side vote limiting" or "file a CR bug that blocks TARS-1381 for the tsquery
crash."

## Arguments

`$ARGUMENTS` — free text describing the ticket: target project (key or name), issue
type (defaults to the project's `defaultIssueType`, else Story), and any link
relationship to another ticket ("blocks X", "follow-up to X", "relates to X").

## Config layout

All configs live in `configs/` beside this skill:

- **`configs/_projects.json`** — project index. `_defaults.cloudId` is the Fandango
  site (`fandango.atlassian.net`); each `<KEY>` entry has `name`, `aliases`,
  optional `defaultIssueType`, and provenance.
- **`configs/<KEY>.json`** — per-project plumbing: `<IssueType>.fields`,
  `<IssueType>.constraints`, and a `transitions` section (used by the
  `jira-transition` skill — leave it intact when editing).

`templates/<IssueType>.md` is content guidance (prose), not config. Never put field
ids in a template or writing rules in a config.

## Workflow

### Step 1: Parse intent

Extract from the prompt:
- **Project** — a key-shaped token ("TOMATORIAL", "CR", "TARS"), a project name, or a
  description ("the editorial project", "critics service"). Resolve the key in Step 2 —
  don't assume a bare uppercase word is already a valid key without checking.
- **Issue type** — explicit ("bug", "story", "task") or inferred from context (a
  crash/defect → Bug; a net-new capability → Story). If neither the prompt nor the
  project's `defaultIssueType` resolves it, ask.
- **Link target + relationship** — a referenced ticket key and the relationship word
  ("blocks", "is blocked by", "relates to", "duplicates"). Map to Jira's link type
  names via `getIssueLinkTypes` if not already known:
  - "blocks" → type `Blocks`, this new ticket is the **inward** issue (blocker),
    the target is the **outward** issue (blocked). "A is blocked by B" → inward=B,
    outward=A.

### Step 2: Resolve the project key

Check `configs/_projects.json` before calling any Jira API:

- If the prompt has an explicit key-shaped token (`[A-Z]{2,}`) that also appears as a
  key in `_projects.json`, use it directly — no lookup needed.
- Otherwise, match the prompt's project name/description against each entry's `name`
  and `aliases`. A hit resolves the key with no live call.
- If a referenced link-target ticket (Step 1) implies the project and neither of the
  above matched, `getJiraIssue` on it already gives you the project key/name for free —
  use that, and still record it below.
- Only if none of the above resolve it, call `getVisibleJiraProjects` with a search
  string built from whatever the user said, resolve the key live, and **append a new
  entry to `_projects.json`**:

```json
{
  "<KEY>": {
    "name": "<project name from Jira>",
    "aliases": ["<the phrase the user actually used, lowercased>"],
    "learnedFrom": "<the phrase or ticket that resolved this>",
    "learnedOn": "YYYY-MM-DD"
  }
}
```

If the key was already in `_projects.json` but the user's phrase isn't yet in its
`aliases`, append the new phrase to the existing `aliases` array (don't overwrite the
entry) so the same wording resolves locally next time too.

Use `cloudId` from the project entry if present, else `_defaults.cloudId`.

### Step 3: Fetch link-target context (if any)

If a ticket was referenced, `getJiraIssue` on it (summary, description, comments).
This is the seed material for Overview/Evidence — same as reading the target ticket
before drafting a follow-up manually. Do not transcribe its full contents into the
new ticket; extract only what the new ticket needs to stand on its own.

### Step 4: Load or discover field requirements

Check `configs/<PROJECT_KEY>.json` (create the file if it doesn't exist — start as
`{}`).

- If `<IssueType>` key exists in the config, reuse its cached `fields` and
  `constraints` — no live metadata call needed.
- If not, call `getJiraIssueTypeMetaWithFields` for the project + issue type, seed a
  new entry under that issue type with every required field (id, name, `fieldType`,
  `allowedValues` if present), and write it back to the config file.

**Config shape:**
```json
{
  "<IssueType>": {
    "fields": {
      "<customfield_id>": {
        "name": "...",
        "required": true,
        "fieldType": "select|multiselect|adf|string|...",
        "allowedValues": [{ "id": "...", "value": "..." }],
        "note": "any quirk worth remembering",
        "learnedFrom": "<ISSUE-KEY>",
        "learnedOn": "YYYY-MM-DD"
      }
    },
    "constraints": [
      { "type": "parentRequired", "note": "...", "learnedFrom": "<ISSUE-KEY>", "learnedOn": "YYYY-MM-DD" }
    ]
  }
}
```

`fieldType: "adf"` is a special case: Jira's create-metadata reports some fields as
plain `textarea`/`string`, but the create/edit endpoint **rejects** a plain string and
requires structured ADF document content (`{ type: "doc", version: 1, content: [...] }`).
Treat any field flagged this way as ADF from the start; don't rediscover it by trial.

### Step 5: Infer parent (if the project needs one)

If `constraints` includes a `parentRequired` entry, or create-metadata otherwise
signals a parent is needed, search recent sibling tickets to find a plausible parent
Epic:

```js
searchJiraIssuesUsingJql({
  jql: `project = <PROJECT_KEY> AND issuetype = <IssueType> ORDER BY created DESC`,
  fields: ["summary", "parent"],
  maxResults: 10,
})
```

Look for a parent Epic shared by multiple recent, topically-related tickets (same
component/domain as the new ticket, not just the most recent one). This is a guess —
surface it as the pre-selected option in Step 7, not a silent decision. **Don't
over-trust "most recent"** as the tiebreaker; match domain/component in the sibling's
summary, and always confirm.

### Step 6: Load content template

Read `templates/<IssueType>.md`. If it doesn't exist, fall back to `templates/Bug.md`'s
shape (Overview / Evidence / Steps-or-equivalent / Acceptance Criteria). After the
ticket is created, offer to save that shape as a new `templates/<IssueType>.md` seed.

Template sections are guidance for **what Claude writes**, not config values — keep
templates as prose/markdown.

### Step 7: One upfront AskUserQuestion

Batch every decision that needs a human into a single `AskUserQuestion` call. No
further prompts after this point. Include, only as applicable:
- **Parent epic** — if inferred in Step 5, present it pre-selected alongside "let me
  specify a different one." **Verify the epic key exists** before creating.
- **Any required field from Step 4 with no default** — surface its `allowedValues`.
- **Ambiguous issue type / link relationship / project match** — if earlier steps
  couldn't resolve confidently.

### Step 8: Generate the description

Using the template from Step 6 and context from Step 3, write each section. Follow the
Philosophy section strictly. Sections that map to Jira fields other than `description`
(check `configs/<PROJECT_KEY>.json`) are written as their own field value, not inlined
into the `description` body.

### Step 9: Create the issue

```js
createJiraIssue({
  cloudId: "<from _projects.json>",
  projectKey: "<PROJECT_KEY>",
  issueTypeName: "<IssueType>",
  summary: "...",
  contentFormat: "markdown",
  parent: "<PARENT_EPIC_KEY>",       // if applicable — pass the key string
  description: "### Overview\n...\n### Acceptance Criteria\n...",
  additional_fields: {
    // one entry per required custom field from configs/<PROJECT_KEY>.json,
    // ADF doc content for any field flagged fieldType: "adf"
  },
})
```

**On a "field is required" error not already in the config** (a constraint the
metadata call didn't surface): ask the user for that value via `AskUserQuestion`, retry
the create, and append the learned constraint/field to `configs/<PROJECT_KEY>.json`
with `learnedFrom`/`learnedOn` set to this ticket, so it's never hit blind again.

### Step 10: Create the link (if any)

```js
createIssueLink({
  cloudId: "<from _projects.json>",
  inwardIssue: "<blocker-key>",
  outwardIssue: "<blocked-key>",
  type: "Blocks",
})
```

### Step 11: Output

```
✅ Created <ISSUE-KEY>: <summary>
🔗 https://fandango.atlassian.net/browse/<ISSUE-KEY>
🔗 Linked: <ISSUE-KEY> blocks <TARGET-KEY>
```

## Guardrails

- **Every issue MUST include Acceptance Criteria.** No exceptions. AC is part of the
  description, written before creating — not deferred to development.
- Never write a plain string into a field whose config marks `fieldType: "adf"` — build
  real ADF doc content.
- Never put an open question or "confirm with @X" into Acceptance Criteria — that's an
  Overview caveat or a separate ticket, not a checkable AC bullet.
- Never transcribe a full investigation/comment thread into the new ticket — summarize;
  link back to the source.
- Every self-healing write to `configs/<PROJECT_KEY>.json` or `_projects.json` must
  include `learnedFrom` and `learnedOn`, so the config stays auditable.
- Never overwrite an existing `_projects.json` entry to add an alias — append to its
  `aliases` array.
- `templates/` = markdown content guidance; `configs/` = JSON plumbing. Don't mix.
- Never call `getVisibleJiraProjects` (or any live project-identity lookup) when
  `_projects.json` already resolves the key or a name/alias match.
- Default to **Story** unless the prompt or the project's `defaultIssueType` says
  otherwise. Verify a parent epic exists before passing it.

## Rationalization Traps

| Excuse | Reality |
|--------|---------|
| "AC can be written after creation" | AC is part of the description. Include before creating. |
| "Description is enough; AC is redundant" | AC = testable goals. Description = context. Both required. |
| "This is a simple fix; doesn't need AC" | Every story needs testable goals, simple or complex. |
| "AC as prose is fine" | Bullet format = scannable, testable. Write bullets. |
| "A bare uppercase word must be the key" | Resolve against `_projects.json` / Jira first. |
| "Most recent sibling's parent is the right epic" | Match domain/component, not recency. Confirm. |
