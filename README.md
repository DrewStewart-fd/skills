# skills

Personal Claude Code skills, kept in one repo and symlinked into the user-level
`~/.claude/skills` directory so every repo on this machine can use them.

## How it works

`~/.claude/skills` is a symlink to this repo:

```
~/.claude/skills -> ~/Projects/skills
```

Claude Code loads user-level skills (`~/.claude/skills/<name>/SKILL.md`) in
**every** project, so any skill added here is immediately available everywhere —
no per-repo setup.

## Adding a skill

1. Create a folder: `~/Projects/skills/<name>/`
2. Add `SKILL.md` with frontmatter:
   ```markdown
   ---
   name: <name>
   description: <when to use this skill>
   ---
   ```
3. Commit. It's live in every repo on next Claude Code session.

## Re-linking on a new machine

```bash
git clone <this-repo> ~/Projects/skills
rm -rf ~/.claude/skills
ln -s ~/Projects/skills ~/.claude/skills
```

## Permissions setup (required for `jira-create-ticket` / `jira-transition`)

Both Jira skills call **write** MCP tools on the `atlassian` server (registered
user-level in `~/.claude.json`, not per-project — confirm with `cat ~/.claude.json`
that an `atlassian` entry exists; if not, the user needs to add it before these
skills work at all).

**Why this matters:** any MCP tool call not pre-listed in `permissions.allow`
triggers an interactive approval prompt. That's fine in a foreground session —
annoying but harmless. It is **not** fine when either skill runs inside a
background subagent (e.g. dispatched via the `Agent` tool, or a `/loop`), because
there's no human present to approve the prompt — the call **auto-denies silently**
and the skill fails partway through (e.g. ticket created but link/transition
silently skipped). Pre-approving the write tools is what makes these skills safe
to delegate to background agents.

**When setting up this repo for a new user/machine, merge this into
`~/.claude/settings.json`** (global user settings — not a project
`.claude/settings.local.json`, since the skills themselves are user-level symlinked
and should work identically in every repo):

```json
{
  "permissions": {
    "allow": [
      "mcp__atlassian__createJiraIssue",
      "mcp__atlassian__editJiraIssue",
      "mcp__atlassian__createIssueLink",
      "mcp__atlassian__transitionJiraIssue",
      "mcp__atlassian__addCommentToJiraIssue",
      "mcp__atlassian__addWorklogToJiraIssue"
    ]
  }
}
```

**Merge, don't replace** — `~/.claude/settings.json` likely already has `model`,
`hooks`, `enabledPlugins`, etc. If a `permissions.allow` array already exists,
append these entries to it rather than overwriting the array. Read the file first.

Read-only calls the skills also make (`getJiraIssue`, `searchJiraIssuesUsingJql`,
`getTransitionsForJiraIssue`, `getIssueLinkTypes`, `getVisibleJiraProjects`,
`getJiraIssueTypeMetaWithFields`) are safe to leave un-approved — worst case is an
extra prompt in foreground use, and background runs of these skills are read-heavy
early, write-only at the end, so the writes above are the only hard blocker.

**Validate after editing:**
```bash
jq -e '.permissions.allow' ~/.claude/settings.json
```
Must exit 0 and print the array. A malformed `settings.json` silently disables
*all* settings in that file, not just permissions — if this fails, fix the JSON
before moving on.
