---
name: writing-for-agents
description: Use when writing or editing a document an agent consumes — a skill, an AGENTS.md / CLAUDE.md, or any doc reached by a pointer. Triggers include "write a skill", "improve this doc", "agent instructions". For opencode config mechanics use opencode-config instead.
---

# Writing For Agents

Packaging differs; the writing levers are the same: make the agent take the
same *process* every run, not the same output.

## Context pointers

A **context pointer** names out-of-context material and encodes when to reach
it: a skill's description, a line in `AGENTS.md` pointing at a doc. Its wording,
not its target, decides when the agent reaches the material. A must-have target
behind a weak pointer is a variance bug: sharpen the wording first, inline only
if sharpening fails.

A pointer does two jobs — say what the material is, and list the **branches**
(distinct cases the doc handles) that trigger reaching it. Every word of an
always-loaded pointer costs on every turn, so prune it hardest:

- **Front-load the leading word** — that's where it fires.
- **One trigger per branch** — synonyms are one branch written twice; collapse them.
- **Cut identity the body already carries.**

## The two loads

Every doc and pointer spends one of two budgets:

- **Context load** — always-loaded material (an `AGENTS.md` line, a skill
  description) costing tokens every turn whether or not it fires.
- **Cognitive load** — the human remembering which docs exist and when to reach
  for each. Not a cost to minimise: it's the price of human agency. Spend it
  where judgement matters, remove it where it doesn't.

## Information hierarchy

Rank content by how immediately the agent needs it:

1. **In-file step** — ordered actions, the primary tier.
2. **In-file reference** — definitions and rules consulted on demand.
3. **Disclosed reference** — pushed to a separate file, reached by a pointer.

**Progressive disclosure** moves material down the ladder so the top stays
legible. Branching is the cleanest test: inline what every branch needs; push
behind a pointer what only some branches reach. **Co-location** keeps a
concept's definition, rules, and caveats under one heading. **Sprawl** is the
failure mode: disclose reference and split by branch or sequence.

## Steps and completion criteria

Every step ends on a **completion criterion** — the observable condition that
tells the agent the work is done. Two levers:

- **Clarity** — can the agent tell done from not-done? A vague bound invites
  **premature completion**: attention slips to *being done*. Sharpen the bound
  first (local, cheap). Only if it's irreducibly fuzzy *and* you observe the
  rush, hide later steps by splitting the sequence — across a real context
  boundary (a handoff or subagent), not an inline call.
- **Demand** — how much it requires. "Every modified model accounted for" forces
  legwork; "produce a change list" doesn't. The strongest criteria are both
  checkable and exhaustive.

## Leading words

A **leading word** is a compact concept the model already holds (*lesson*,
*fog of war*, *tracer bullets*). Repeated as a token, never a sentence, it
anchors a region of behaviour in few tokens by recruiting priors. A coined word
recruits no priors — reach for an existing one first.

It anchors twice: in the body, **execution** (same behaviour every time the word
appears); in a pointer, **invocation** (the shared word links prompt, doc, and
code). Collapse spelled-out triads into one token: "fast, deterministic,
low-overhead" → *tight*.

**Negation** is the failure mode beside it: steering by prohibition drags the
banned behaviour into context and makes it *more* available. Prompt the
**positive** — state the target ("write one-line comments") so the banned one is
never spoken. A prohibition earns its place only as a hard guardrail you can't
phrase positively; even then, pair it with the positive target.

## Pruning

- **Single source of truth.** One authoritative place per meaning. Duplication
  costs maintenance and inflates prominence past its real rank.
- **The environment is a source of truth too.** `package.json` scripts, config,
  directory layout — a doc restating them is a cache earning its load only when
  the lookup is expensive. Cache what the agent can't find by looking: the
  unwritten convention, the reason behind a choice, the gotcha no config admits.
- **Check relevance line by line.** Stale layers settle because adding feels
  safe and removing feels risky. Shorter docs stay relevant.
- **Hunt no-ops.** An instruction the model obeys by default pays load to say
  nothing. Test: does it change behaviour vs the default? If not, delete the
  whole sentence, not words from it.

Source: mattpocock/skills (MIT) — adapted and trimmed.
