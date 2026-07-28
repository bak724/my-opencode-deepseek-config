---
name: diagnose
description: Structured 6-phase debugging loop for hard bugs and performance issues. Use when the task mentions "diagnose", "debug", "trace", "find root cause", "排查", "诊断", or reports a bug that resists first-attempt fixes. Complementary to systematic-debugging (superpowers) — use diagnose when you need a tighter feedback loop.
license: MIT
compatibility: opencode
metadata:
  audience: developers
  workflow: debugging
---

# Diagnose

A 6-phase structured debugging loop. Core thesis: a tight pass/fail signal finds
root cause faster than reading code. If you don't have one, don't start yet.

REQUIRED BACKGROUND: Load superpowers:systematic-debugging first for complex
bugs — it frames the investigation; this skill provides the execution loop.

## Phase 1: Build the feedback loop

If you have no pass/fail signal, code-reading is guessing. Options (fastest first):
- Failing test at any seam touching the bug
- Curl/HTTP script against running dev server
- CLI invocation with fixed input, diff against known-good snapshot
- Headless browser script (Playwright/Puppeteer)
- Git bisect harness

Loop must be: (a) red when bug exists, green when fixed; (b) deterministic;
(c) fast (seconds); (d) agent-runnable (unattended).

HARD RULE: If you catch yourself forming theories before the loop exists — STOP.
Hypothesis-before-loop is the failure mode this skill prevents.

## Phase 2: Reproduce + Minimize

Confirm the loop reproduces the exact symptom. Minimize: cut inputs, callers,
config, data, steps one at a time, re-running after each. Stop when every
remaining element carries load (removing any one turns the loop green).

## Phase 3: Hypothesize

Generate 3-5 ranked, falsifiable hypotheses. Format: "If <X> is the cause, then
<changing Y> makes the bug disappear / <changing Z> makes it worse." Show the
ranked list to the user before testing — domain knowledge may re-rank instantly.

## Phase 4: Instrument

Each probe maps to one Phase 3 prediction. Change ONE variable at a time.
Priority: debugger > targeted logging > never "log everything and grep".
Tag debug logs with unique prefix `[DEBUG-a4f2]` — cleanup is one grep.
For performance: baseline (timer/profiler), then bisect.

## Phase 5: Fix + Regression test

Write the regression test BEFORE the fix — but only if a proper seam exists
(the test exercises the real bug pattern as callers actually hit it). If no
proper seam: document the gap — the architecture prevents bug lock-in.

If seam exists: 1) minimized repro to failing test, 2) watch it fail, 3) fix,
4) watch it pass, 5) re-run Phase 1 loop against original scenario.

## Phase 6: Cleanup + Post-mortem

- Original repro no longer reproduces
- Regression test passes (or seam gap documented)
- All `[DEBUG-...]` instrumentation removed; throwaway prototypes deleted
- Correct hypothesis stated in commit/PR message
- Ask: what would have PREVENTED this? If architectural (no test seam,
  entangled callers, hidden coupling), escalate to codebase improvement.

## Rules

- Phase 1 before Phase 3. Always. No loop = you're guessing.
- One variable at a time in Phase 4.
- Regression test first in Phase 5.
- No `[DEBUG-...]` lines left behind in Phase 6.
