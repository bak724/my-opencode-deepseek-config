---
name: tech-debt-audit
description: Audit a codebase for technical debt and report a prioritized, read-only findings list. Use when the user asks to audit tech debt or code health, or the task mentions "tech debt", "技术债", "code audit", "audit", "refactor", "debt".
---

# Tech Debt Audit

Read-only code health assessment. Produce a prioritized report grouped by
dimension with `file:line` evidence and a minimal fix per finding. Never modify
code — audit only.

## Dimensions

Scan all nine; skip any with no findings rather than padding.

1. **Dead code** — unused files, exports, functions, variables, imports; code
   behind feature flags no longer reachable.
2. **Duplication** — copy-pasted blocks (6+ identical lines in 2+ places);
   logic that diverges across copies.
3. **Naming / convention drift** — identifiers or patterns that diverge from the
   codebase's established conventions; catch-all files.
4. **Complexity** — deep nesting (3+ levels), long functions, god objects,
   multiple responsibilities in one unit.
5. **Dependencies** — redundant, outdated, or unused packages; version drift vs
   the lockfile.
6. **Error handling** — empty catch blocks, swallowed errors, missing
   propagation, `@ts-ignore` without reason.
7. **Tests** — missing coverage for core logic, fragile tests (timing, snapshots
   of unstable output), tests that assert nothing.
8. **Docs** — stale, missing, or misleading docs/comments; docs that contradict
   the code.
9. **Security** — injection, hardcoded secrets, unsafe deserialization, path
   traversal. Surface here; route deep review to the `security-review` skill.

## Method

1. Map the repo first — use the `codemap` skill for orientation; reuse
   `remove-deadcode` phase-1 scanners to surface dead-code candidates.
2. Gather evidence per dimension: file, line, and why it is debt.
3. Prioritize by blast radius and fix cost, not by raw count.

## Report format

```markdown
## Tech Debt Audit — <repo>

### 1. Dead code
- [high] `src/foo.ts:42` — unused export `Bar` → remove or wire up.

### 2. Duplication
- ...
```

Group by dimension; within each, order by priority (`high`/`medium`/`low`). Each
finding: priority, `file:line`, one-line issue, and a minimal fix.

End with a short summary: total findings per dimension and the three
highest-leverage fixes. Do not apply any fix — report only.
