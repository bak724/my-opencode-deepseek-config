---
name: to-questionnaire
description: Use when a decision or set of facts can't be answered from the conversation and must be collected from someone off-channel as a one-off questionnaire filled in async — not for a live requirements interview (that's grilling). Triggers include "questionnaire", "survey", "ask the stakeholder", "what should I ask X".
---

# To Questionnaire

Turn something the user can't answer alone into a **questionnaire** — a Markdown
doc handed to one person to fill in async or over a meeting. The recipient holds
knowledge the user lacks.

**Grill the send, not the subject.** Interview the user only about the *send*
(which they can always answer): who it goes to, what they need back. The
questions then target the **gap** between what the recipient knows and what the
user needs.

1. **Who is it going to?** One exchange: recipient's role, expertise,
   relationship. Fixes tone and context.
2. **What do you need back?** One exchange: the concrete decisions or facts the
   user can't resolve alone.
3. **Write the questionnaire.** Draft questions aimed at that gap, write to
   `to-questionnaire-<slug>.md`, report the path.

## Document structure

Frame it as a **discovery questionnaire**. Order questions most-important-first
(async = one pass); group under `##` by theme once there are more than a
handful:

```
# <Title>

**Purpose:** why this exists and the decision riding on it.

**From:** <user> — **To:** <recipient> — **How answers are used:** <where they go>

## Context
One paragraph orienting a recipient who wasn't in the user's head.

## How to answer
Deadline and effort. Partial answers and "I don't know" are useful.

## <Theme>
Questions under this theme, most-important-first. One idea per question, never
compound, with an answer stub beneath and a one-line *why this matters* only
where the question could be misread.

## Anything else?
A closing catch-all.
```

Source: mattpocock/skills (MIT) — adapted and trimmed.
