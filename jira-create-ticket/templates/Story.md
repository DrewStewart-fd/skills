<!--
Philosophy for every section: this ticket is the baseline a human or
agent picks up cold. Write only what's known — never pad with a full
investigation dump. Be concise, concrete, and actionable. Someone
reading this should immediately know what the feature is and what
"done" looks like; they'll do their own research from here.

Open questions (things that need a person's judgment, not a test) do
NOT belong in Acceptance Criteria — put them in Overview as a caveat,
or file them as a separate follow-up. Every AC bullet must be
independently checkable with no extra context.

Summary rules (the ticket title, not part of this body):
- One line, ~50 chars, specific. Verb or noun phrase.
- No "Fix"/"Add" prefix (action is clear from context).
- ✅ "Server-side vote limiting for rt-polls"
- ❌ "Fix rt-polls" / "Add a feature"
-->

### Overview
{1-3 sentences: what the feature/change is and why it matters. Plain
language. If there's an unresolved question or caveat, state it HERE —
not in Acceptance Criteria.}

### Scope
<!-- Omit if the Overview already makes scope obvious. Keep to 1-2 bullets. -->
- {in scope}
- {explicitly out of scope, if worth stating}

### Acceptance Criteria
<!--
REQUIRED — never omit. Plain bullet list (conditions aren't sequential).
Each bullet is one testable, unambiguous condition, NOT a task or an
implementation detail.

✅ "User cannot vote more than once per poll"
✅ "Server returns 429 on duplicate vote attempt"
❌ "Implement vote limiting" (task, not testable goal)
❌ "Create a votes_per_user column" (implementation detail)
❌ "Tests pass" (vague — say which behavior is verified)
-->
- {testable condition}
- {testable condition}
