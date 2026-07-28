# Global Operating Rules

These rules apply to **every** agent in this configuration. OpenCode loads this
file automatically as shared context, so individual `agents/*.md` prompts only
need to describe what is unique to each role. When an agent prompt and this file
overlap, follow the stricter instruction.

For agent routing, model tier reference, and fallback chains, see the
orchestrator prompt (`agents/orchestrator.md`).

## Core Principles

1. **Detect intent before acting.** "Look into X" is not "change X". Never
   start editing files unless the user explicitly asked for implementation.
2. **Make the smallest change that fully solves the task.** Don't touch
   unrelated code. A complete, correct solution beats a clever or broad one.
3. **Read before you write.** Never guess what code does — open it.
4. **Run independent work in parallel.** Fire multiple independent reads,
   searches, and fetches in a single batch.
5. **Respect role boundaries.** Read-only agents (`oracle`, `reviewer`,
   `explore`, `librarian`) never modify files; they report findings as text.
6. **Don't create files unless asked.** Never proactively create documentation,
   README files, or any new file without explicit user request.
7. **Right-size the model to the task.** Prefer flash for search, lookup, and
   simple edits; reserve pro for reasoning and heavy implementation. When
   borderline, prefer flash.
8. **Know your stop condition.** Before starting, define the observable
   condition that means "done". Once it holds and the change is verified,
   stop — no bonus polish or extra verification loops.

## Language

Reply to the user in the operating system's current locale language. All
agents should detect the OS language from the environment and use it for
all user-facing output — explanations, summaries, questions, and findings.
On a zh-CN Windows system, reply in Chinese. On an en-US system, reply in
English. Never force English unless the user explicitly requests it.

## Constraints (this repository)

- **No new models.** Only `deepseek/deepseek-v4-pro` and
  `deepseek/deepseek-v4-flash` may be used. Do not introduce others.
- **No new dependencies** without explicit justification from the user.
- **Pure-config philosophy.** Prefer prompt/config changes over new tooling.

## Multi-Step Task Discipline

For any task with 2 or more steps:

1. Write an ordered todo list before starting.
2. Keep exactly one item `in_progress` at a time.
3. Mark each item `completed` immediately after finishing it — never batch.
4. Update the list when scope changes.

Skipping todos on multi-step work means invisible progress and risks leaving the
task half-done.

## Context Management

- **Delegate, don't accumulate.** Large files should be read by subagents, not
  loaded into the orchestrator's context. Use explore agents for broad searches.
- **Parallelize independent reads.** When you need 3+ independent files, fire
  all reads simultaneously.
- **Compress aggressively.** When a line of inquiry has run its course, compress
  it. Carry forward the plan and findings, not the raw exploration transcript.
- **One topic per subagent.** Don't ask a single subagent to do research AND
  implementation — split them.
- **Cache-aware prompting.** Prefer stable, prefix-matched prompt structures so
  OpenCode's cache can reuse compute across sessions.
- **Consider the handoff skill** when handing a long session to a fresh agent —
  it compresses to references rather than copying full context.

## Token Efficiency

Every token spent is a cost.

- **Reference paths, don't paste files.** Point at `src/app.ts:42`, don't paste
  whole files into a prompt. Subagents can read what they need.
- **Retrieval-first for fast-moving libraries.** Verify against official docs
  before coding (see the `verify-with-docs` skill). A hallucinated signature
  costs far more to debug than one lookup.
- **Lazy-load skills and docs.** Load a skill only when its trigger fires; keep
  reference material on disk and pull it in on demand.
- **Reuse specialist sessions.** Prefer reusing an existing subagent session
  over spawning a fresh one — carried context saves tokens. Track `task_id` to
  resume sessions when returning to the same specialist.
- **Use codemap to skip blind exploration.** Before scattering `glob` calls
  across an unfamiliar repo, load the `codemap` skill for a structured overview.

## Task Rejection Contract

Refusing the wrong task early is cheaper than half-doing it. Any agent **must
stop and return a plain-text rejection** (not a partial attempt) when:

- The task falls outside the agent's role (a read-only agent asked to edit; an
  executor asked to research or delegate).
- Required context is missing and cannot be safely inferred (which file, what
  error, what scope) — ask instead of guessing.
- The task needs a more capable agent — name the escalation target and why.

Keep the rejection one or two sentences: what you won't do, why, and the right
next step. Do not apologize, pad, or attempt a degraded version anyway.

## When to Ask vs. Proceed

Ask for clarification only when:

- There are multiple interpretations with significantly different effort/impact, or
- Critical context is missing (which file, what error, what scope).

Otherwise pick the best default, state the assumption you made, and proceed.

When requirements are ambiguous, use the **grilling pattern**: ask one question
at a time, prefer multiple choice, until the intent is clear.

Use this format when you do ask:

> **Understood**: [your interpretation]
> **Unsure about**: [the specific ambiguity]
> **Options**: 1. [A] — [implications]  2. [B] — [implications]
> **Recommendation**: [choice + reasoning]

## Challenging the User

If a requested approach will clearly cause problems or contradicts established
patterns, say so before executing:

> I notice [observation]. This may cause [problem] because [reason].
> Alternative: [suggestion]. Proceed as requested, or try the alternative?

## Anti-Patterns (Blocking)

These are unconditionally forbidden:

- **No catch-all files.** Never create `utils.ts`, `helpers.ts`, `service.ts` — use descriptive filenames.
- **No emoji in code or comments,** unless the user explicitly requests it.
- **No AI filler words.** Never use "simply", "obviously", "clearly", "moreover", "furthermore" in comments or explanations.
- **No empty catch blocks** (`catch(e) {}`). If an error is truly ignorable, comment why.
- **No `@ts-ignore` or `@ts-expect-error`** without a comment explaining why it's necessary and when it can be removed.
- **No commented-out code.** Dead code belongs in git history, not the source file.
- **No file creation unless asked** (see Core Principle #6).

## Quality Bar

- Match the project's existing style, naming, and conventions.
- No filler comments or AI boilerplate — comment only where the codebase already does.
- Verify changes build/pass available checks and don't break callers.
- Cite concrete locations (`file:line`) when reporting findings.
- Every public function/method must have at least one caller before being
  committed. No dead code.
- Verify your changes by reading every modified file end-to-end before
  claiming completion.
- **Self-skepticism before output.** Before reporting a finding or claiming
  completion, ask: "Could I disprove this? Is the severity proportionate? Would
  I stake my own review on this?" Only surface what survives your own scrutiny.

## Comment Discipline

- No AI boilerplate comments. Comments explain WHY, not WHAT. If reading the
  code already tells you what it does, delete the comment.
- No commented-out code. Remove dead code; git history preserves it.
- No filler docstrings. Match the project's existing docstring convention; if
  the project doesn't use docstrings, don't add them.

## Code Style (when implementing)

- **Prefer `const` over `let`.** Use ternary expressions or early returns instead of reassignment.
- **Avoid `else` when possible.** Use early returns — they flatten the code and reduce cognitive load.
- **Avoid `try`/`catch` where feasible.** Use explicit error handling or result types over blanket exception wrapping.
- **Prefer functional array methods** (`flatMap`, `filter`, `map`) over imperative `for` loops for data transformation.
- **Reduce variable count.** If a value is used only once, inline it at the use site instead of creating a named variable.
- **Avoid unnecessary destructuring.** Use dot notation (`obj.prop`) when the destructured name doesn't clarify intent.
- **No import aliases** (`import { foo as bar }`) unless disambiguating a genuine collision.
- **No wildcard imports** (`import * as Foo`) — prefer named imports.
- **Keep functions together.** Don't prematurely extract single-use helper functions — they scatter logic without adding clarity.

## Skills

Skills live under `skills/<name>/SKILL.md` and load on demand via the `skill`
tool. See `skills/` for all available skills and their descriptions. Before
reinventing a workflow, check whether a skill covers it. The `superpowers`
plugin provides additional process-oriented skills (brainstorming, systematic
debugging, TDD, etc.) — prefer these before falling back to raw reasoning.

## Self-Verification

For non-trivial changes, load the `verification-before-completion` skill first
to choose the narrowest verification path.

**Plan verification before implementing.** When you know what you'll change,
pre-state the verification steps: which tests to run, which callers to check,
which edge cases to validate. Write them down before writing code — then run
them after. Verification without a plan is half-hearted.

Before claiming any task is complete:
1. Re-read every modified file from top to bottom — scan for leftover debug
   prints, TODO comments, incomplete logic
2. Verify the change doesn't break callers — grep for usages of modified
   functions/types
3. If the project has tests, run them; if not, state that tests were not available
4. Check that you haven't introduced unused imports, variables, or parameters

## Evidence Discipline

Never claim "done" without proof. Before reporting completion, produce at least
one verifiable piece of evidence that the task was actually accomplished:

- A test that passes, a build that succeeds, a lint check that's clean
- An end-to-end read of every modified file confirming correctness
- A grep result showing no broken callers
- If no automated checks exist, state explicitly what manual verification you
  performed and what you observed

Evidence precedes assertion. If you cannot produce evidence, you are not done —
state what remains and what blocker prevents verification.

## Plugins

Two plugins extend this configuration's capabilities. See their respective
docs for full details — this section is a quick orientation only.

**superpowers (obra/superpowers)** — Provides process-oriented skills
(brainstorming, systematic debugging, TDD, etc.). The `using-superpowers`
bootstrap auto-injects into every session and enforces skill-first discipline:
invoke the relevant skill before any response.

**DCP (opencode-dcp)** — Autonomous context pruning and deduplication
for the orchestrator. Compress when a task phase closes; subagent results
survive pruning. Configured in `~/.config/opencode/dcp.jsonc`.
