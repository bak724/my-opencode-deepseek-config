---
name: writing-great-skills
description: Use when creating new skills, editing existing skills, or the task mentions skill/write-skill/SKILL.md/writing-skills. Covers no-op trimming discipline, positive phrasing, completion criteria, and embedded Conventional Commits format reference.
---

# Writing Great Skills

How to write and maintain high-quality, token-efficient skills for OpenCode.

## The No-Op Trim (Most Important)

Run this test on every sentence in every skill:

> **If I delete this sentence, does the agent's behavior change?**

If the answer is **no**, delete the sentence immediately. Agents already know
the default behaviors you're describing. Every sentence you keep must pass this
test.

## Positive Phrasing

Say what the agent SHOULD do, never what it should NOT do.

```
BAD:  "Don't forget to read the file before editing."
GOOD: "Read the file before editing."
BAD:  "Never skip the verification step."
GOOD: "Run the verification step after every change."
```

Negatives ("don't skip", "avoid forgetting") make the forbidden behavior more
accessible to the model. Frame every instruction as a positive action.

## Completion Criteria

Every step in a skill MUST have an observable, checkable completion condition.

```
BAD:  "Review the code for issues."
GOOD: "Read the file from top to bottom. List every issue found, with file:line."
```

Steps without completion criteria cause premature termination — the agent
declares "done" without actually finishing. A completion criterion is a
condition anyone can check: "file exists at path X", "grep returns 0 matches",
"output contains the string Y".

## Information Hierarchy

- **In-step content** — what the agent MUST know to execute this step
- **In-skill reference** — background info the agent MIGHT need
- **External reference** — links to docs the agent fetches on demand

Push everything possible to external references. Keep in-step content to the
minimum needed for correct execution.

## Trim Checklist

Before finalizing any skill, scan for:

1. **No-op sentences** — deleted without behavior change → delete
2. **Repetition** — same instruction appears twice → keep once
3. **Bloated descriptions** — 20 words where 5 suffice → compress
4. **Negative phrasing** — "don't X" → "do Y"
5. **Vague steps** — no completion criterion → add one

## Conventional Commits Format (Reference)

When writing commit messages, use this format:

```
type(scope): imperative-mood subject

Types: feat / fix / docs / chore / refactor / test / style / perf / ci
BREAKING CHANGE: description (footer, when applicable)
```

The subject uses imperative mood ("add" not "added"), lowercase, no trailing period.
