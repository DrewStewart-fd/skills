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
