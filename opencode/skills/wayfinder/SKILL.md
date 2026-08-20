---
name: wayfinder
description: Navigate the fog of a large, unfamiliar codebase before any plan. Use when starting work in a huge/legacy/multi-repo codebase, when the task is a multi-step feature with unknown scope, or when the task mentions "too big to hold in context", "where do I even start", "fog of war", "map the unknown", or "navigate a large codebase". Produces a decision-ticket map tracked in a local Markdown file; resolves one ticket per session.
---

# Wayfinder

Orient inside a codebase too large to hold in context, then drive toward the
goal one bounded step at a time — without a plan, and without boiling the ocean.

## When to use

- The codebase is huge, legacy, multi-repo, or you cannot see the whole shape.
- The goal is a multi-step feature whose exact scope is still unknown.
- You keep re-reading files because there is no durable map of what is known.

Not for small codebases (use `codemap`), not for single fixes (use `explore`).

## Core loop (one session resolves ONE ticket)

1. **Take stock** — list what you know and what you do not (the "fog of war").
2. **Pick ONE decision ticket** from the frontier — the highest-value unknown
   whose resolution unblocks the most downstream work.
3. **Resolve it** with the cheapest tool that answers it (research, prototype,
   grilling, or a task).
4. **Record the outcome** in the tracker; push newly-discovered unknowns onto
   the frontier; mark blocking edges.
5. **Stop** after one ticket. Do not chain-resolve in a single pass.

## The decision-ticket map

A local Markdown tracker (default `wayfinder/tickets.md` in the repo root; do
NOT bind to an issue tracker). Four ticket kinds:

| Kind | Question it answers | Resolved by |
|---|---|---|
| `research` | What is here / how does it work? | reading code, docs, `codemap` |
| `prototype` | Will this approach work? | throwaway spike (see `prototype` skill) |
| `grilling` | What does the user actually want? | `grilling` skill, one question at a time |
| `task` | A small, bounded piece of work | `deep-worker` / direct edit |

Each ticket carries:
- `goal` — one sentence: the decision to make.
- `kind` — one of the four above.
- `blocking` — tickets that must resolve first (directed edges).
- `status` — `open` | `in-progress` | `resolved` | `wontfix`.

## Tracker format

````markdown
# Wayfinder tickets — <goal in one line>

## Frontier (open)
- [ ] T1 (research) Where is request routing wired? — blocks: T3
- [ ] T2 (grilling) Confirm target users — blocks: none
- [ ] T3 (task) Add rate-limit header — blocks: T1

## Resolved
- [x] T0 (research) Repo layout & build entry — resolved by codemap

## Fog of war (known unknowns)
- Auth flow: seen but not traced.
- Deploy pipeline: unread.
````

## Rules

- **One ticket per session.** Resolving more than one means the tickets were
  too small — merge them next time.
- **Frontier over depth.** Prefer tickets that unblock the most edges; never
  tunnel on a dead end.
- **Fog of war is first-class.** A recorded unknown is progress, not failure.
- **Never invent a plan from fog.** If the frontier is not yet clear, resolve
  one more research ticket before proposing anything.
- **Track in the repo, not chat.** The map must survive the session and stay
  greppable; reference it by path from then on.
- **Cite evidence.** Each resolved ticket links `file:line` or the artifact
  that settled it.
