---
name: code-review
description: Token-frugal, multi-dimension code review for a diff/branch/PR. Use when reviewing changes, checking a PR, running a review→fix loop, or the task mentions "code review", "review my changes", "review this PR", "找 bug", "审查代码". Scales review depth to diff size, reports findings by dimension and severity, and calibrates against the project's threat model to avoid severity inflation. Reports findings; never rewrites code unless explicitly asked.
---

# Code Review

A structured, token-frugal review discipline: one pass covers all dimensions,
depth scales to diff size, and large reviews communicate through files instead
of context. Pair with `security-review` when the diff touches a trust boundary.

## Step 0 — Scope by *effective* size, not raw lines

Never review blindly. Establish the change set before reading code:

- Branch vs base: `git diff --stat main...HEAD` (or the stated base)
- PR: use the `gh-cli` skill (`gh pr diff <n> --patch`, `gh pr view <n>`)
- Explicit files: just those paths

Then weight each changed file by category — a 2000-line lockfile diff is not a
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

**High-stakes override** — route **Full** regardless of size when any logic/config
file touches auth/authz, DB migrations or schema, concurrency/locking, or a
public API/wire contract. These are where a missed finding costs the most.

Abbreviated is the default — it costs ~an order of magnitude fewer tokens.
State which path you took, the effective size, and any stakes trigger in one line.

## Dual-Axis Review (two parallel passes)

For standard and full reviews, dispatch TWO parallel sub-review passes:

1. **Standards axis** — style, naming, structure, comments, imports, error
   handling. Check against AGENTS.md Anti-Patterns and Code Style rules.
2. **Spec axis** — correctness, boundary conditions, security, performance.
   Check against the stated task/feature requirements.

Run both passes concurrently. After both complete, merge their findings:
deduplicate (same issue found by both), sort by severity, produce a single
report with the axis source tagged on each finding.

### Standards Axis Checklist (inline — pass with task dispatch)

Check every changed file against:
- [ ] Style: const/let usage, early returns, functional array methods
- [ ] Naming: no catch-all files, descriptive filenames, no import aliases
- [ ] Structure: no wildcard imports, no commented-out code, no empty catch
- [ ] Comments: why not what, no AI filler words, no emoji
- [ ] Imports: named imports only, no unused imports, no alias
- [ ] Error handling: no empty catch, no @ts-ignore without comment

### Spec Axis Checklist (inline — pass with task dispatch)

Check every changed function/logic path against:
- [ ] Correctness: Does it do what the requirements say?
- [ ] Boundaries: null/undefined, empty arrays, zero values, max values
- [ ] Security: injection, auth, secrets, path traversal, deserialization
- [ ] Performance: N+1 queries, unnecessary loops, large allocations
- [ ] Concurrency: race conditions, shared mutable state (if applicable)

## Mechanical scan (before dimension review)

A quick scan for mechanical issues:

- **Duplicates:** Any block of code (6+ lines) that appears verbatim in 2+ places
  within the diff? Flag as potential copy-paste.
- **Pattern drift:** Does this change follow the same patterns as adjacent code
  (naming, error-handling style, file structure)? Flag deviations.
- **Naming mismatch:** Do new identifiers use consistent terminology with the
  rest of the file? Flag synonyms used for the same concept.
- **Dead imports:** Any new import that has no usage in the added code? Flag.

Report mechanical findings in a single block before dimensions.

## Review dimensions

Cover every dimension that the diff actually touches. Skip dimensions with no
relevant changes rather than padding the report.

1. **Correctness** — logic bugs, off-by-one, null/undefined, unhandled edge
   cases, error paths.
2. **Security** — injection, XSS, authz/authn gaps, secrets, path traversal,
   SSRF, unsafe deserialization. Load `security-review` skill if any apply.
3. **Performance** — N+1 queries, unbounded loops/allocations, blocking calls
   on hot paths, missing pagination/timeouts, leaks.
4. **Architecture** — inappropriate coupling, leaky abstractions, wrong-layer
   responsibility, needless complexity.
5. **Maintainability** — naming, function size, magic numbers, dead code,
   duplicated logic, convention drift.
6. **Docs & comments** — Enforce AGENTS.md Comment Discipline and Anti-Patterns
   (commented-out code, AI boilerplate, stale docs).
7. **Compatibility** — breaking API/signature changes, altered public contracts,
   changed defaults, DB/schema migrations, callers left unupdated.

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
- Prefer one accurate high-severity finding over ten inflated ones — every false
  alarm erodes trust.
- If a whole category consistently doesn't apply here, say so once and suggest
  recording it in AGENTS.md.

### Suppress known-design noise

Treat documented decisions (from caller's context note, `.opencode/decisions.md`,
or `AGENTS.md`/`CLAUDE.md`) as intentional. Flag only when the diff makes a
documented choice concretely unsafe (e.g. trusted-input assumption now reachable
from an untrusted path).

## Calibration

Before assigning severity, read `.ai/calibration.yml` if it exists. For any
finding whose pattern matches a calibration entry, apply the specified
downgrade and cite the entry in the report. This prevents the same
low-priority finding from being flagged as high in every review.

## Self-skepticism check (before output)

Before writing any finding, silently run this adversarial check. Default to
**rejection** — a finding earns its place only by surviving scrutiny:

1. **Could I disprove this?** Build a counter-argument. "This is fine because…
   the error handler on line N already covers this / it only fires on admin
   paths / the input is validated upstream at line M." If the counter-argument
   is stronger than the finding, discard it.
2. **Is the severity inflated?** Would the severity hold up under a second
   reviewer's scrutiny? If you have to stretch to justify "critical," it's not
   critical. Downgrade by one level if unsure.
3. **Is this a real issue or a preference?** "This variable name could be
   better" is not a finding unless it causes real confusion. Style preferences
   that the project doesn't enforce are not review items.

**Reject a finding immediately if any of these conditions apply** (borrowed from
deepreview's validator rejection rules, condensed):

- The cited `file:line` is wrong or the code isn't actually in the diff's blast
  radius.
- It targets **pre-existing, unchanged** code this diff didn't touch (note it as
  context at most — it is not a review item for this change).
- The severity is inflated relative to the project's threat model (see
  calibration above).
- It is a pure design/style **opinion** the project doesn't enforce.
- It **duplicates** another finding — merge, don't list twice.
- It is a documented, intentional decision (see suppression above).

Only surface findings that survive all three questions and none of the rejection
rules.

Before confirming any finding, write one sentence arguing why it might be wrong,
irrelevant, or not worth fixing. If the counterargument is stronger than the
finding, downgrade or dismiss it.

Lead with a one-line severity summary:
`critical: N | major: N | minor: N | nit: N` and the path taken (abbreviated/full).

Then list findings, ordered by severity, each as:

```
[severity] <title>  (dimension, confidence: high|medium)
<!-- id: <12-char-hash> -->  (SHA-256 of file:line:title, for loop dedup)
location: path/to/file.ext:LINE
issue:  <what is wrong and the input/condition that triggers it>
impact: <what breaks, or what an attacker/user gains>
fix:    <the minimal concrete remediation a flash agent could apply>
```

Tag each finding's **confidence** (high = verified in code, severity
proportionate; medium = plausible but not fully traced). Lead with high-confidence
findings within each severity level.

Close with a short overall assessment (merge-ready? blocking items?) and a brief
**What Looks Good** line naming the parts that are solid — it tells the author
what not to second-guess and signals you actually read the change. If the change
is genuinely clean, say so plainly — do not manufacture findings.

### Doc drift batching

Non-critical documentation findings (severity != critical) are batched into a
single `## Document Drift` section as a checklist, not scattered across the
report. Only surface a docs finding as its own entry when it is genuinely
dangerous (a false claim that could cause API misuse, a security-critical
misleading comment).

### Large reviews — communicate through files

On the Full path, write the findings block to a file (e.g.
`.opencode/review-<short-ref>.md`) and return only the severity summary plus the
file path to the caller. This keeps large review content out of the
orchestrator's context — the single biggest review token cost.

**Response contract for file-based reviews:** after writing the file, your reply
is *only* the summary line + the absolute path — nothing else. Do not restate
findings in the chat; the file is the artifact. (File-IPC contract;
deepreview's multi-agent pipeline is omitted.)

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

Stop conditions: **clean** (no findings above nit), **max 5 iterations**, or
**convergence**. Classify each re-review finding as **NEW**, **RECURRING**
(unresolved from last pass), or **REGRESSION** (reintroduced by a fix). Keep
looping only while a pass produces NEW or REGRESSION findings above nit severity;
once a pass yields zero such findings, stop and surface any remaining
RECURRING findings for a human rather than thrashing on fixes that aren't
converging. (Novelty-based convergence; deepreview's multi-agent pipeline
omitted.)

On iterations 2+, prepend a `## Prior Findings` block listing findings from the
previous pass (with their IDs). Do NOT re-report a prior finding unless it is
a REGRESSION (was fixed in the interim, now broken again).

### Convergence check (for review→fix loops)

Use the novelty tags from the Review → fix loop to drive convergence:

- **converging**: 0 NEW findings, or NEW < previous round's NEW
- **deadlocked**: 0 NEW but RECURRING findings persist across 2+ rounds
- **diverging**: NEW > previous round's NEW

Stop conditions:

- **Clean:** No critical or major findings remain
- **Converging + clean enough:** Only minor/nit findings and 0 NEW in last round — surface to user
- **Deadlocked:** Same RECURRING findings appear 2+ rounds despite fixes — the fix approach is wrong; pause and re-assess
- **Diverging:** Regressions are being introduced — stop immediately, report
- **Hard cap:** 5 iterations maximum — force-stop and report

Track iteration count and novelty breakdown at each round. Report:
`Round N: NEW=X, RECURRING=Y, REGRESSION=Z, severity: critical=A, major=B. Verdict: [converging|deadlocked|diverging|clean]. Next: [continue|stop|re-assess]`

## Posting to a PR

To publish findings on GitHub (e.g. `/review-pr`), use the `gh-cli` skill.
`gh pr review` posts **only** a top-level verdict + body (`--approve` /
`--comment` / `--request-changes` with `--body`); it has **no** flag for per-line
comments. To attach findings to specific lines, post a single pending review via
the REST API:

```bash
gh api repos/{owner}/{repo}/pulls/<n>/reviews --method POST \
  -f event=COMMENT -f body="severity summary…" \
  -F 'comments[][path]=src/app.ts' -F 'comments[][line]=42' \
  -F 'comments[][body]=<!-- cr:auth-nullcheck-L42 --> issue…'
```

Use `event=COMMENT` (never auto-`APPROVE` — leave the verdict to a human). Line
numbers must be the **new-side** line inside a changed hunk.

Place each finding at the tightest scope its location allows (3-tier placement):

1. **Line comment** — the finding's `file:line` falls inside a changed hunk.
2. **File-level comment** — the file is in the diff but the line is outside any
   hunk (e.g. a caller you had to widen to).
3. **Review body** — the finding is about a file not in the diff at all
   (cross-cutting or blast-radius concerns).

This keeps comments anchored to the diff. Triage: lines within diff hunks → line
comment; lines in diff files but outside changed blocks → file comment; files
not in the diff → review body only.

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
