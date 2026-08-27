<!--
QA Notes content guidance. Used by the jira-transition skill at the
Code Review transition, and by any project whose config marks a QA
Notes field as fieldType: "adf".

QA Notes is almost always an ADF field — a plain string is rejected
("Operation value must be an Atlassian Document"). Wrap the text in an
ADF doc. For step lists, use an orderedList of listItem → paragraph →
text nodes.
-->

# QA Notes heuristic

Decide what QA can actually test, then write for that reader:

- **User-facing / testable behavior changed** → write numbered QA test
  steps a tester can follow with no extra context. Include real
  URLs/paths so they can copy-paste and go. State the expected result
  for each meaningful step.
- **Nothing user-facing to test** (infra cleanup, dead-code removal,
  tooling, refactors with no behavior change) → set QA Notes to
  **"Covered by regression."**

## ADF wrapper (plain-text case)

```json
{
  "type": "doc",
  "version": 1,
  "content": [
    { "type": "paragraph", "content": [{ "type": "text", "text": "Covered by regression." }] }
  ]
}
```

## ADF wrapper (step list)

```json
{
  "type": "doc",
  "version": 1,
  "content": [
    {
      "type": "orderedList",
      "content": [
        { "type": "listItem", "content": [
          { "type": "paragraph", "content": [{ "type": "text", "text": "Go to <url>." }] }
        ]},
        { "type": "listItem", "content": [
          { "type": "paragraph", "content": [{ "type": "text", "text": "Expected: <result>." }] }
        ]}
      ]
    }
  ]
}
```
