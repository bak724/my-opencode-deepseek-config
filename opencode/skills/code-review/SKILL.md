---
name: code-review
description: Token-frugal, multi-dimension code review for a diff/branch/PR. Use when reviewing changes, checking a PR, running a review→fix loop, or the task mentions "code review", "review my changes", "review this PR", "找 bug", "审查代码". Scales review depth to diff size, reports findings by dimension and severity, and calibrates against the project's threat model to avoid severity inflation. Reports findings; never rewrites code unless explicitly asked.
---

# Code Review

A structured, token-frugal review discipline: one pass covers all dimensions,
depth scales to diff size, and large reviews communicate through files instead
of context. Pair with `security-review` when the diff touches a trust boundary.

## Step 0 — Scope by *effective* size, not raw lines

Establish the change set before reading code:

- Branch vs base: `git diff --stat main...HEAD` (or the stated base)
- PR: use the `gh-cli` skill (`gh pr diff <n> --patch`, `gh pr view <n>`)
- Explicit files: just those paths

Weight each changed file by category — a 2000-line lockfile diff is not a
2000-line review. Sum the **effective logic lines**:

| File category | Weight | Examples |
| --- | --- | --- |
| generated / mechanical | **0×** | lockfiles, `*.pb.go`, snapshots, `@generated`, import-only reshuffles |
| data / config | **0.25×** | `.json`, `.yaml`, `.toml`, `.tf`, fixtures |
| tests | **0.5×** | `*_test.*`, `*.spec.*`, `__tests__/` |
| logic | **1×** | everything else (source code) |

Pick the path from effective size **and** stakes:

| Condition | Path | Behavior |
| --- | --- | --- |
| **≤ 8 logic files and ≤ 300 effective lines** | **Abbreviated** (default) | Single focused pass over the diff and its immediate callers. Report inline. |
| **larger** | **Full** | Walk each dimension deliberately; write findings to a file (see below). |

**High-stakes override** — route **Full** regardless of size when any changed
logic/config file matches: `auth|authz|migration|schema|lock|concurr|public api|
wire contract|serializ`. A hit upgrades the depth; no hit keeps the default.

Abbreviated is the default — it costs ~an order of magnitude fewer tokens.
State which path you took, the effective size, and any stakes trigger in one line.

## Review dimensions (dual-axis)

Dispatch TWO parallel passes covering all dimensions the diff touches. Skip
dimensions with no relevant changes — don't pad.

### Standards axis — style, naming, structure, comments, imports, error handling

Check against AGENTS.md Anti-Patterns and Code Style rules:

1. **Maintainability** — naming (no catch-all files, descriptive filenames, no
   import aliases), function size, magic numbers, duplicated logic, convention
   drift, no wildcard imports, no commented-out code.
2. **Docs & comments** — why not what, no AI filler words, no emoji, no stale
   docs. Enforce AGENTS.md Comment Discipline and Anti-Patterns.
3. **Compatibility** — breaking API/signature changes, altered public contracts,
   changed defaults, DB/schema migrations, callers left unupdated.

### Spec axis — correctness, boundaries, security, performance

Check against the stated task/feature requirements:

1. **Correctness** — logic bugs, off-by-one, null/undefined, unhandled edge
   cases, error paths. No empty catch, no `@ts-ignore` without comment.
2. **Security** — injection, XSS, authz/authn gaps, secrets, path traversal,
   SSRF, unsafe deserialization.
3. **Performance** — N+1 queries, unbounded loops/allocations, blocking calls
   on hot paths, missing pagination/timeouts, leaks.
4. **Architecture** — inappropriate coupling, leaky abstractions, wrong-layer
   responsibility, needless complexity, race conditions (if applicable).

Run both passes concurrently. After both complete, merge: deduplicate (same
issue found by both), sort by severity, tag each finding with its axis source.
A finding reported independently by both axes is highest confidence — tag it
`highest-confidence`.

### Mechanical scan (before dimension review)

Quick scan before the dual-axis passes:

- **Duplicates:** 6+ identical lines in 2+ places within diff → flag as copy-paste.
- **Pattern drift:** new code doesn't follow adjacent patterns (naming, error
  handling, file structure) → flag deviation.
- **Naming mismatch:** new identifiers use different terms for same concept → flag.
- **Dead imports:** new import with no usage in added code → flag.

Report mechanical findings in one block before the dimensional review.

Before reporting, silently verify: read every changed file end-to-end; check
unused imports, leftover TODOs, debug prints; confirm new functions have callers.

## Severity levels

- **critical** — data loss, security hole, crash, or broken core behavior. Must fix.
- **major** — real bug or regression under plausible input; wrong results.
- **minor** — narrow-impact bug, weak error handling, notable smell.
- **nit** — style/naming/comment polish. Report only if it compounds into a
  maintainability problem; otherwise omit.

## Severity calibration (fight inflation)

Judge impact in context against the project's actual threat model and conventions.
Read `package.json` or equivalent to detect project stage (v0.x vs v1+), deployment
model (localhost tool? public service? internal tool? library?), and repo visibility.

- Read AGENTS.md (and CLAUDE.md if present) for stated context, threat model, and
  conventions before assigning severity.
- Apply these heuristics:
  - **v0.x projects**: API stability/compatibility findings → minor at most
    (semver expects breaking changes).
  - **Localhost-only tools**: auth/network attack surface → minor (documented
    constraint).
  - **Internal tools**: external attack vectors → minor.
  - **v1+ public libraries**: API breaks, unvalidated input → critical/major.
- Down-rank findings that don't apply to this project's reality. Note the
  calibration reason.
- Prefer one accurate high-severity finding over ten inflated ones.
- If a whole category consistently doesn't apply here, say so once and suggest
  recording it in AGENTS.md.

### Suppress known-design noise

Treat documented decisions (from caller's context note, `.opencode/decisions.md`,
or `AGENTS.md`/`CLAUDE.md`) as intentional. Flag only when the diff makes a
documented choice concretely unsafe.

## Calibration

Before assigning severity, read `.ai/calibration.yml` if it exists. For any
finding whose pattern matches a calibration entry, apply the specified
downgrade and cite the entry in the report. This prevents the same
low-priority finding from being flagged as high in every review.

## Validator pass (before output)

Default to **rejection** — every finding must survive scrutiny. Verify each
finding against four checks before writing it:

1. **Falsifiability** — build a counter-argument. If the counter ("it only fires
   on admin paths / input is validated upstream at line M") is stronger than the
   finding, discard it.
2. **Severity** — would this hold under a second reviewer? Downgrade if unsure.
3. **Preference vs defect** — style opinions the project doesn't enforce are not
   review items.
4. **Evidence** — the citation must be correct and inside the diff's blast radius.

This pass discards **disproved** findings outright: wrong `file:line`, code
outside the blast radius, pre-existing unchanged code, severity beyond the
threat model, style opinions stated as defects, duplicates, or documented
intentional decisions.

## Report format

Lead with a one-line severity summary:
`critical: N | major: N | minor: N | nit: N` and the path taken (abbreviated/full).

Then list findings, ordered by severity, each as:

```
[severity] <title>  (dimension, class: confirmed|plausible)
<!-- id: <12-char-hash> -->  (SHA-256 of file:line:title, for loop dedup)
location: path/to/file.ext:LINE
issue:  <what is wrong and the input/condition that triggers it>
impact: <what breaks, or what an attacker/user gains>
fix:    <the minimal concrete remediation a flash agent could apply>
```

Classify each finding: **confirmed** (verified at the cited line, severity
proportionate) or **plausible** (likely but not fully traced). Lead with
confirmed findings within each severity level. **trivial** findings (real but
cosmetic) go to Document Drift, not the main list.

Close with a short overall assessment (merge-ready? blocking items?) and a brief
**What Looks Good** line naming the parts that are solid. If the change is
genuinely clean, say so plainly — do not manufacture findings.

### Doc drift batching

Non-critical documentation findings are batched into a single `## Document Drift`
section as a checklist, not scattered across the report. Route **trivial**
findings here too. Only surface a docs finding as its own entry when it is
genuinely dangerous (a false claim that could cause API misuse, a
security-critical misleading comment).

### Large reviews — communicate through files

On the Full path, write the findings block to a file (e.g.
`.opencode/review-<short-ref>.md`) and return only the severity summary plus the
file path to the caller. This keeps large review content out of the
orchestrator's context — the single biggest review token cost.

**Response contract for file-based reviews:** after writing the file, your reply
is *only* the summary line + the absolute path — nothing else.

## Review → fix loop

When asked to review *and fix* (e.g. `/review-loop`), run a bounded loop:

1. Review the current diff (scope + dimensions + severity as above).
2. If no findings above `nit`, stop — report clean.
3. Apply the minimal fixes for `critical`/`major` findings (and clear `minor`
   ones). Follow AGENTS.md; keep changes surgical.
4. **Verify**: run the project's format/lint/test commands. Discover them from
   `AGENTS.md` first, else infer from the repo (`package.json` scripts,
   `Makefile`, `mise.toml`, `cargo`, `ruff`, etc.). If none exist, state that
   verification was skipped.
5. Re-review only the changed region. Repeat.

Classify each re-review finding as **NEW**, **RECURRING** (unresolved from last
pass), or **REGRESSION** (reintroduced by a fix). On iterations 2+, prepend a
`## Prior Findings` block listing findings from the previous pass (with their
IDs). Do NOT re-report a prior finding unless it is a REGRESSION.

Stop by novelty and hard cap:

- **Clean**: no critical/major findings remain.
- **Converging + clean enough**: only minor/nit and 0 NEW in the last round — surface to the user.
- **Deadlocked**: same RECURRING findings 2+ rounds despite fixes — the fix approach is wrong; pause and re-assess.
- **Diverging**: regressions being introduced — stop immediately, report.
- **Hard cap**: 5 iterations maximum — force-stop and report.

Track iteration count and novelty at each round. Report:
`Round N: NEW=X, RECURRING=Y, REGRESSION=Z, severity: critical=A, major=B. Verdict: [converging|deadlocked|diverging|clean]. Next: [continue|stop|re-assess]`

## Posting to a PR

To publish findings on GitHub (e.g. `/review-pr`), load the `gh-cli` skill.
`gh pr review` posts only a top-level verdict; per-line comments require the
REST API — see the `gh-cli` skill ("Reviewing PRs" section) for the `gh api`
pattern. Use `event=COMMENT` (never auto-`APPROVE`). Line numbers must be the
**new-side** line inside a changed hunk.

Place each finding at the tightest scope its location allows (3-tier placement):

1. **Line comment** — the finding's `file:line` falls inside a changed hunk.
2. **File-level comment** — the file is in the diff but the line is outside any
   hunk.
3. **Review body** — the finding is about a file not in the diff at all.

**Re-review dedup:** embed a stable finding id (`<!-- cr:auth-nullcheck-L42 -->`)
in each comment body. On later passes, list existing review comments (`gh pr view
<n> --json comments` or `gh api`), match by id, and update in place. Add only new
findings; resolve or note ones the diff has since fixed.

## Rules

- Report findings; do not modify code unless in loop mode.
- Review the diff and blast radius first; widen only when needed.
- Follow AGENTS.md quality and comment rules.
- Cite concrete `file:line` locations.
- No performative positivity or inflated severity.
