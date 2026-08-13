---
name: shared-language
description: Use when the task mentions domain language, CONTEXT.md, terminology, shared vocabulary, or when the same concepts are being explained repeatedly in different sessions. Builds and maintains a shared language document.
---

# Shared Language

Build and maintain `.opencode/CONTEXT.md` — a glossary of project-specific
terms and concepts, each defined in ONE sentence. Every conversation that
explains the same concept in long-form wastes tokens. A shared language
document replaces those explanations with a single reference.

## When to Create or Update

1. You find yourself explaining the same concept (>2 sentences) across
   multiple messages or sessions.
2. You encounter a term the user uses that has project-specific meaning.
3. After a design discussion — capture the decisions as shared language
   entries.

## Format

`.opencode/CONTEXT.md` uses this exact format:

```markdown
# Shared Language

- `<term>`: <one-sentence definition>
- `<term>`: <one-sentence definition>
```

Each entry is **one line**. No paragraphs, no examples, no "further reading".

## How Agents Use It

Before reasoning about the codebase, scan `.opencode/CONTEXT.md`:

1. If a term appears, use its one-sentence definition directly — do not
   re-explain or expand.
2. If a concept is missing but you needed 3+ sentences to convey it, add an
   entry after the session.

The goal: each term's definition replaces 5-20 sentences of repeated
explanation across sessions. The agent also spends fewer tokens on thinking
because the concepts are pre-compressed.

## Maintenance

- Keep entries to one sentence. If an entry grows, it's a sign it should be
  a design doc instead.
- Delete entries that no longer apply (removed features, renamed concepts).
- Sort alphabetically.
