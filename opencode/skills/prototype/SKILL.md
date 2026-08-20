---
name: prototype
description: Answer a design question with a throwaway, one-day prototype instead of a spec or a long conversation. Use when a decision hinges on "will this feel right", "does this interaction work", "is this layout better", or the task mentions "spike", "quick prototype", "visualize the options", "try it out before committing". Produces a runnable HTML demo (logic) or multiple style variants (UI); never persists, never polishes.
---

# Prototype

Settle a design question empirically with the cheapest runnable artifact,
then throw it away. A prototype is evidence, not product code.

## When to use

- A logic/behavior question: "does this flow work end-to-end?"
- A UI question: "which layout/style variant is better?"
- You are about to write a spec or have a long back-and-forth over a decision
  that a 5-minute demo would settle.

Not for: building real features (use `deep-worker`), or questions answerable
by reading (use `explore`).

## The two kinds

| Question type | Build |
|---|---|
| **Logic / behavior** | a single self-contained HTML file with inline JS that interactively demonstrates the logic end-to-end |
| **UI / style** | one route with 2–3 style/layout variants rendered side by side (or toggled), so the choice is visible, not argued |

## Iron rules

- **One day, one use.** The artifact is thrown away; do not extend or maintain it.
- **One command to run.** `open prototype.html` (or equivalent); no build step,
  no dependencies, no framework.
- **No persistence.** No backend, no DB, no localStorage (unless the prototype
  is about persistence itself).
- **Skip polish.** No design system, no tests, no error handling beyond what
  the demo needs. Fake data is fine.
- **Show state after every action.** After each interaction, render the current
  state so the behavior is observable, not inferred.

## Workflow

1. State the design question in one sentence.
2. Pick the kind (logic vs UI).
3. Build the single HTML file in the working tree.
4. Run it (open in browser); confirm it demonstrates the answer.
5. Show the result to the user; capture the decision.
6. Commit to a throwaway branch as the primary source (e.g.
   `git checkout -b throwaway/prototype-<topic>`), then discard after the
   decision is recorded.

## Rules

- Never let a prototype silently become production code.
- If the question is still open after the demo, that is the answer: it was the
  wrong question — rephrase, do not extend the prototype.
- Keep the artifact self-contained; no external CDN unless strictly required.
- Record the decision (and the prototype's path) in the relevant plan/spec,
  not in the prototype.
