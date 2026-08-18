---
name: wizard
description: Use when a human must manually walk through steps only they can perform — provisioning infrastructure, setting up credentials or CI secrets, navigating an unfamiliar dashboard, or running a one-off migration or cutover. Not for steps the agent can do itself. Triggers include "setup wizard", "walk me through", "provision", "secrets", "migration".
---

# Wizard

A **wizard** is a bash script that walks a human step by step through a manual
procedure: opens each URL, says what to click and copy, captures values, writes
them where they belong (`.env`, GitHub secrets), confirms each stage, and shows
progress. Ephemeral by default — one run, then delete. Commit only for a
repeatable setup path.

## Process

### 1. Scope the procedure

Work out every manual step and every captured value. Read the repo first, don't
ask cold:

- Setup: `.env`, `.env.example`, `README`, `docker-compose*`, framework config,
  and `.github/workflows/*` — every `secrets.*` / `vars.*` reference is a value
  the wizard must produce.
- Migration/transition: current state, target state, and the irreversible
  actions between.

Show the user the ordered stage list with the values each produces, and confirm
(add, drop, reorder).
**Done:** every stage named in order; for each captured value you know (a) where
the human gets it, (b) where it's written (`.env`, a secret, both, nowhere), and
(c) whether it's secret.

### 2. Map each stage's journey

Write the precise path: which URL, what to do there, where a value shows, which
variable it fills. Where you don't know the current UI or
exact command, ask or check the docs — never invent steps.
**Done:** every stage traces to instructions a stranger could follow.

### 3. Author the wizard

Write one self-contained bash script. Above a `STAGES` marker, put the library
block — identical in every wizard, never hand-edited:
`stage` (clear screen + progress), `say`/`step` (instructions), `open_url`
(cross-platform incl. WSL), `ask`/`ask_secret` (hidden entry), `write_env`
(idempotent `.env` upsert), `set_secret`/`set_var` (via `gh`), `pause`/`confirm`,
and `finish` (closing summary). Below the marker, one `stage` per step in
dependency order, and set `TOTAL_STAGES` to match.

Hold the bar: `open_url` before asking for its value, `ask_secret` for anything
secret, `write_env` every persisted value, `set_secret` only what CI needs, and
`confirm` before any irreversible action. Keep a stage to one focused task (each
clears the screen).

### 4. Verify and hand off

- `bash -n <script>`; run `shellcheck` if available. `chmod +x <script>`.
- Don't run it end-to-end — it opens browsers and blocks on input. Trace
  statically: every value from step 1 is captured and lands where step 1 said;
  every `set_secret` name matches a `secrets.*` reference in CI.
- Tell the user how to run it. For a repeatable path, commit it and link it from
  the README.

Source: mattpocock/skills (MIT) — adapted and trimmed.
