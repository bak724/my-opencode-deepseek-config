---
name: spec-workflow
description: Run a lightweight, spec-driven change workflow — capture WHY/WHAT in a proposal, define WHAT the system does in specs, decide HOW in a design, break work into a tasks checklist, implement, then archive. Use when the task is a non-trivial feature or behavior change that benefits from durable, git-tracked planning artifacts, or when the request mentions "spec", "proposal", "propose a change", "spec-driven", "openspec", "design doc", "requirements", or "tasks checklist". Not for one-line fixes or pure Q&A.
license: MIT
compatibility: opencode
metadata:
  audience: developers
  workflow: spec-driven
---

# Spec-Driven Change Workflow

A lightweight way to plan and land a change through durable, git-tracked
artifacts that both humans and coding agents share. Adapted (pure prompt, no
external tooling) from the OpenSpec OPSX workflow — this skill does **not**
require the `@fission-ai/openspec` npm package, only the conventions below.

## Philosophy

- **Enablers, not gates.** Artifact dependencies (proposal → specs → design → tasks)
  show what's possible next, not what you MUST do. You can always jump back and
  edit an earlier artifact — discovered the design is wrong mid-implementation?
  Update `design.md` and keep going.
- **Fluid, not rigid.** These are actions you take anytime — not locked phases.
  Implement, discover the design was wrong, update the spec, keep going.
- **Artifacts are the source of truth.** Plans live as markdown files in the
  repo, not just in chat, so they survive the session and stay greppable.
- **Brownfield-friendly.** Works on existing codebases; research current
  behavior before proposing changes. Delta specs (ADDED/MODIFIED/REMOVED/RENAMED)
  mean you describe only what changes, not the entire system — ideal for
  modifying an existing codebase without rewriting every spec.
- **Scale down.** Skip the whole ceremony for a one-line fix. Use the ephemeral
  `planner` Handoff Plan when no durable artifact is needed. Reach for this
  workflow when a change spans multiple files/sessions or needs a shared spec.

## Directory convention

```text
openspec/
├── specs/                       ← source of truth: current, shipped behavior
│   └── <capability>/spec.md
├── explorations/                ← optional: pre-proposal thinking notes
│   └── <topic>.md
└── changes/
    ├── <change-id>/             ← one in-flight change (kebab-case id)
    │   ├── proposal.md          ← WHY + WHAT (required)
    │   ├── design.md            ← HOW (optional, only when warranted)
    │   ├── tasks.md             ← checklist the apply step tracks (required)
    │   └── specs/<capability>/spec.md   ← delta: what this change adds/edits
    └── archive/
        └── YYYY-MM-DD-<change-id>/       ← completed changes, kept for history
```

Project-wide context and rules (tech stack, conventions) live in `AGENTS.md` —
treat it as this repo's equivalent of a project config, and honor it in every
artifact.

## Action: explore (optional, before propose)

A no-stakes thinking pass **before** committing to a proposal — OpenSpec's
`/explore` command. Use it when the direction is still fuzzy: research the
current behavior, sketch options and trade-offs, and surface open questions,
*without* creating `changes/<change-id>/` yet. Keep it in chat (or a scratch note
under `openspec/explorations/<topic>.md` if it needs to persist). The goal is to
avoid locking a proposal onto a premature direction — jump to `propose` only once
the shape is clear.

## Action: propose

Create `openspec/changes/<change-id>/` with a proposal, tasks, delta specs, and
(when warranted) a design. Research existing code and `openspec/specs/` first.

**proposal.md** — keep it to 1–2 pages, focus on WHY not HOW:

```markdown
# <change-id>

## Why
1–2 sentences: the problem or opportunity. Why now?

## What Changes
- Specific new capabilities, modifications, or removals.
- Mark breaking changes with **BREAKING**.

## Capabilities
- New: `<capability>` — becomes `specs/<capability>/spec.md`
- Modified: `<existing-capability>` — needs a delta spec

## Impact
Affected code, APIs, dependencies, or systems.
```

Include a `## Updates` section at the bottom of `proposal.md` (initially empty)
in the template above to track future revisions.

**tasks.md** — the apply step parses `- [ ]` checkboxes, so follow this exactly:

```markdown
## 1. <group>
- [ ] 1.1 <small, verifiable task>
- [ ] 1.2 <task>

## 2. <group>
- [ ] 2.1 <task>
```

Order tasks by dependency; each should fit in one session and be verifiable.

**design.md** — create **only** if any apply: cross-cutting change, new
architectural pattern, new dependency, significant data-model change, or
security/performance/migration complexity. Sections: Context · Goals/Non-Goals ·
Decisions (with alternatives considered) · Risks/Trade-offs · Migration Plan ·
Open Questions.

**Open Questions** — problems the design intentionally defers. Each entry names
what is unknown, why it can wait, and what would trigger revisiting it. If an
open question would change the specs, approach, or task breakdown, resolve it
now (ask the user) — do not guess and defer.

## Writing spec files

Specs define WHAT the system does and are testable. Each scenario is a potential
test case.

- Requirement header: `### Requirement: <name>` followed by a description using
  **SHALL/MUST** (avoid "should"/"may" for normative behavior).
- Scenario header: `#### Scenario: <name>` — **exactly 4 hashtags**, then
  WHEN/THEN bullets. A scenario written with 3 hashtags (`###`) or as a plain
  bullet instead of a `####` header is silently ignored (not tracked as a
  scenario).
- Every requirement MUST have at least one scenario.

Delta specs (files under a change's `specs/`) group edits by operation header:

- `## ADDED Requirements` — brand-new requirements.
- `## MODIFIED Requirements` — copy the **entire** existing requirement block,
  then edit it; partial content loses detail when archived.
- `## REMOVED Requirements` — include **Reason** and **Migration**.
- `## RENAMED Requirements` — `FROM:`/`TO:` only.

```markdown
## ADDED Requirements

### Requirement: User can export data
The system SHALL allow users to export their data as CSV.

#### Scenario: Successful export
- **WHEN** the user clicks "Export"
- **THEN** the system downloads a CSV with all their data
```

## Action: apply

Read the proposal, design, and specs for the change, then work through
`tasks.md`: pick the next unchecked task, implement it, mark it `- [x]`, repeat.
Pause and ask if you hit a blocker or the design proves wrong (update the
artifacts rather than silently diverging).

## Action: update

Revise an existing proposal in-place when the user wants to modify an
already-proposed change that has not yet been archived:

1. Read `openspec/changes/<change-id>/proposal.md`, `design.md` (if it exists),
   and `tasks.md`.
2. Understand the requested update — what should change and why.
3. Mark tasks that are affected by the update as `[ ]` (unchecked) if they were
   already completed.
4. Add new tasks for the requested change.
5. Update the delta specs to reflect the new scope.
6. Annotate changes in `proposal.md` with a brief "Updated: YYYY-MM-DD — [reason]"
   entry in the `## Updates` section.
7. Report what was changed and what tasks need re-execution.

## Update vs New Change Heuristic

When asked to update an existing change, apply this test before modifying
the artifacts:

- **Update** the existing change when: same intent AND scope overlap >50%
  (the change addresses substantially the same problem). OR when the
  existing change cannot realistically be "completed" as-is (wrong
  direction, stale assumptions).

- **Create a new change** when: the intent has shifted (different problem
  statement) OR scope has exploded (3x+ original). Think of this like git
  branches — a small fixup amends the branch; a direction change creates
  a new branch.

## Action: verify (before archive)

Before archiving, confirm the change is actually complete. This catches half-done
work and spec-implementation drift before it becomes the source of truth:

1. **Task completion**: every checkbox in `tasks.md` is `[x]`. If any are
   unchecked, stop — the change is not complete. Do not archive with open tasks.
2. **Spec coverage**: for each `## ADDED`/`## MODIFIED` requirement in the delta
   spec, can you point to the code that fulfills it? If a requirement has no
   corresponding implementation, flag it.
3. **Consistency**: do the proposal's "What Changes" claims match the actual
   implementation? If the implementation diverged from the spec, update the
   artifacts (use the `update` action) before continuing.

Report findings in three tiers: **CRITICAL** (must fix before archive — open
tasks, missing implementation), **WARNING** (spec-implementation mismatch that
needs an artifact update), **SUGGESTION** (untracked improvements worth noting).

Do not block archive for WARNING/SUGGESTION — just surface them so the human can
decide. CRITICAL means archive is not safe yet.

## Action: archive

After verify passes (no CRITICAL findings), when all tasks are done and merged:

1. **Merge delta specs into the source of truth.** For each delta spec file under
   `changes/<change-id>/specs/<capability>/spec.md`:
   - `## ADDED` → append new requirements to `openspec/specs/<capability>/spec.md`.
   - `## MODIFIED` → replace the existing requirement block with the full updated
     version (delta specs must contain the **entire** modified block, not fragments).
   - `## REMOVED` → delete from the source of truth, but verify the **Reason** and
     **Migration** are recorded before removal.
   - `## RENAMED` → rename the requirement header (keep content, update `FROM:`/`TO:`).
2. Move `openspec/changes/<change-id>/` to
   `openspec/changes/archive/YYYY-MM-DD-<change-id>/`.
3. Verify `openspec/specs/` now reflects reality and the change dir is clean.
4. Commit the archive with a message like `docs(openspec): archive <change-id>`.

If a capability spec file doesn't exist yet under `openspec/specs/`, create it
and seed it with the ADDED requirements from the delta.

## Anti-patterns

- Ceremony for a trivial fix — use `planner`'s ephemeral Handoff Plan instead.
- MODIFIED specs with partial content (loses detail at archive time).
- Scenarios with 3 hashtags or bullet form (silently untracked).
- Implementing past the tasks list without updating the artifacts.
