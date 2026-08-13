# My OpenCode × DeepSeek Config

[简体中文](README.md) | [繁體中文](README.zh-TW.md) | **English** | [Русский](README.ru-RU.md) | [Français](README.fr-FR.md) | [Deutsch](README.de-DE.md) | [Español](README.es-ES.md) | [Português](README.pt-BR.md) | [日本語](README.ja-JP.md) | [한국어](README.ko-KR.md)

**OpenCode × DeepSeek Optimal Config** — A configuration scheme designed to maximize the capabilities of the DeepSeek V4 dual-model (Pro + Flash) within the OpenCode multi-agent framework. Core philosophy: **Token efficiency first, achieving the best development results with the minimum context cost**.

## Current Configuration Overview

- Default main Agent: `orchestrator`
- Main model: `deepseek/deepseek-v4-pro`, lightweight model: `deepseek/deepseek-v4-flash`
- Agent depth: `subagent_depth: 3` (supports 3 levels of agent nesting)
- Model isolation: `enabled_providers: ["deepseek"]` + `disabled_providers` double lock
- Session sharing: disabled (`share: "disabled"`); snapshots: enabled (`snapshot: true`)
- Permission baseline: allow by default, destructive bash commands set to `ask`; `.env`-like sensitive files set to `deny`; external directories set to `ask`; bash whitelist for read-only Agents (deny all by default + allow only read-only subcommands)
- Context compression: DCP 60% threshold proactive compression + OpenCode native auto compaction near-overflow fallback, two complementary layers (prune trims old tool outputs)
- Global rules: `AGENTS.md` (core principles, task rejection contract, self-verification, anti-patterns, etc.; context/token discipline delegated to `orchestrator`)
- Skills: **18** `SKILL.md` skills in the `skills/` directory, loaded on-demand via the native `skill` tool
- Plugins: `superpowers` (v6.3.0, process-type skills), `@tarquinen/opencode-dcp` (intelligent context trimming)

## DeepSeek Model Configuration

### Prerequisites

- OpenCode ≥ v1.14.24 (DeepSeek provider is built-in)
- DeepSeek API Key: Apply at [platform.deepseek.com/api_keys](https://platform.deepseek.com/api_keys)

### Method 1: TUI Interactive Configuration (Recommended)

```bash
opencode
# In TUI enter: /connect → select DeepSeek → paste API Key
# Then: /models → select deepseek-v4-pro
```

The API Key will be automatically persisted to `~/.local/share/opencode/auth.json`.

### Method 2: Environment Variables

Windows PowerShell:
```powershell
$env:DEEPSEEK_API_KEY="sk-your-key-here"
opencode
```

Permanent setup: Add `DEEPSEEK_API_KEY` to your system environment variables.

### Provider Configuration Reference

```jsonc
{
  "model": "deepseek/deepseek-v4-pro",
  "small_model": "deepseek/deepseek-v4-flash",
  "enabled_providers": ["deepseek"],
  "disabled_providers": ["openai", "anthropic", "google", "openrouter"]
}
```

To enable thinking/reasoning for the Pro model, append to `provider`:

```jsonc
"provider": {
  "deepseek": {
    "models": {
      "deepseek-v4-pro": {
        "options": {
          "thinking": { "type": "enabled" }
        }
      }
    }
  }
}
```

> **Model ID Naming Convention**: `provider_id/model_id`, i.e., `deepseek/deepseek-v4-pro` and `deepseek/deepseek-v4-flash`.

## Installation & Deployment

### Method 1: Clone + Environment Variable (Recommended, Cross-Platform)

```bash
git clone https://github.com/znlgis/my-opencode-deepseek-config.git
```

Then set `OPENCODE_CONFIG_DIR` to point to the `opencode/` subdirectory within the repo.

**Windows (PowerShell)** — permanent:

```powershell
[Environment]::SetEnvironmentVariable("OPENCODE_CONFIG_DIR", "D:\path\to\my-opencode-deepseek-config\opencode", "User")
```

**Windows (PowerShell)** — temporary (current session only):

```powershell
$env:OPENCODE_CONFIG_DIR = "D:\path\to\my-opencode-deepseek-config\opencode"
opencode
```

**Linux / macOS** — append to `~/.bashrc` or `~/.zshrc`:

```bash
export OPENCODE_CONFIG_DIR="$HOME/path/to/my-opencode-deepseek-config/opencode"
```

### Method 2: Symlink to Global Config Directory

**Windows (PowerShell, requires Administrator):**

```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.config"
New-Item -ItemType SymbolicLink -Path "$env:USERPROFILE\.config\opencode" -Target "D:\path\to\my-opencode-deepseek-config\opencode"
```

**Linux / macOS:**

```bash
ln -s /path/to/my-opencode-deepseek-config/opencode ~/.config/opencode
```

> **Compatibility Note**: `~/.config/opencode` is the standard global config path for OpenCode. The `opencode/` subdirectory of this repository contains `agents/`, `skills/`, `AGENTS.md`, and other files whose layout fully follows OpenCode conventions. It will be automatically recognized after pointing to it via an environment variable or symlink.

### Verify Installation

Start OpenCode and verify:
1. `/models` → Current model is `deepseek/deepseek-v4-pro`
2. Agent list should show `orchestrator`, `planner`, `deep-worker`, etc. (10 Agents total)
3. Enter any request; the Orchestrator automatically analyzes intent and routes

## Model Division of Labor

This repository strictly confines its division of labor to the DeepSeek V4 dual-models, without introducing external models:

| Model | Usage |
| --- | --- |
| `deepseek/deepseek-v4-pro` | Planning, architecture, root cause analysis, code review, heavy implementation, main orchestration |
| `deepseek/deepseek-v4-flash` | Rapid exploration, external retrieval, lightweight tasks, simple edits |

### Routing Strategy

- **Flash First**: Explicitly defined tasks like searching, locating, and simple editing prioritize the flash agent
- **Pro for Reasoning**: Planning, analysis, review, complex implementations—use only pro
- **Automatic Upgrade**: Automatically upgrades to pro when the flash agent cannot handle the task (with full context)

## Agent Structure

### Primary Agent

| Agent | Model | Role |
| --- | --- | --- |
| `orchestrator` | v4-pro | Default entry: Intent Gate + Model-aware routing + Fallback chain |

### Subagents

| Agent | Model | Permissions | Role |
| --- | --- | --- | --- |
| `planner` | v4-pro | Read/Write | Planning, architecture, task breakdown |
| `deep-worker` | v4-pro | Read/Write | Heavy implementation, multi-file changes, complex debugging |
| `oracle` | v4-pro | **Read-Only** | Root cause analysis, deep code understanding |
| `reviewer` | v4-pro | **Read-Only** | Dual-axis code review (specifications + conventions) + severity calibration |
| `ui-builder` | v4-pro | Read/Write | Frontend & UI related tasks |
| `consultant` | v4-pro | Read/Write | Solution discussion, best practice advice |
| `explore` | v4-flash | **Read-Only** | Codebase search, parallel exploration |
| `librarian` | v4-flash | **Read-Only** | Documentation retrieval, Web search |
| `light-orchestrator` | v4-flash | Read/Write | Lightweight tasks, single-file editing |

> `deep-worker` and `light-orchestrator` follow the "no research, no delegation" principle—they execute rather than explore, with context provided by the orchestrator.
>
> Read-only Agents (`oracle`/`reviewer`/`explore`/`librarian`) are truly read-only: `edit: deny` + bash whitelist (deny all by default, allow only read-only subcommands such as `git status/diff/log/show/blame/grep` and `rg`; `oracle`/`reviewer` additionally allow `gh pr view/diff`, `gh issue view`, `gh api` to support `/review-pr` replies).

## Quick Commands

### Agent Routing Commands

| Command | Agent | Usage |
| --- | --- | --- |
| `/deep` | `deep-worker` | Heavy implementation, multi-file changes |
| `/quick` | `light-orchestrator` | Lightweight tasks, single-file editing |
| `/ui` | `ui-builder` | Frontend/UI work |
| `/review` | `reviewer` (code-review) | Dual-axis parallel review (specifications + conventions) + severity calibration |
| `/review-pr` | `reviewer` (code-review + gh-cli) | Review PRs and reply to them on GitHub |
| `/plan` | `planner` | Create plans, technical solutions |
| `/search` | `librarian` | External search, documentation lookup |
| `/oracle` | `oracle` | Deep analysis, problem tracing |
| `/consult` | `consultant` | Consulting, comparisons, recommendations |

### Operation Commands

| Command | Agent | Usage |
| --- | --- | --- |
| `/commit` | `light-orchestrator` | Generate Conventional Commits messages (inline format) |
| `/release` | `deep-worker` (git-release) | Prepare Tag releases |
| `/reflect` | `oracle` (reflect) | Identify friction → propose config optimizations |
| `/handoff` | `light-orchestrator` (handoff) | Compress session into handoff documents |

### Inline Commands

| Command | Agent | Usage |
| --- | --- | --- |
| `/codemap` | `explore` (codemap) | Generate repository structure map |
| `/simplify` | `oracle` (simplify) → `light-orchestrator` | oracle analyzes → light-orchestrator applies simplification |
| `/rmslop` | `deep-worker` (remove-deadcode) | Clean up dead code and AI slop |

### Spec Commands

| Command | Agent | Usage |
| --- | --- | --- |
| `/spec-propose` | `planner` (spec-workflow) | Explore code → draft change proposals |
| `/spec-apply` | `deep-worker` (spec-workflow) | Implement step-by-step via tasks.md → auto-archive |

## Skills

OpenCode exposes skills on-demand via the native `skill` tool—Agents only load them when needed, saving context.

| Skill | Role |
| --- | --- |
| `code-review` | Token-frugal multi-dimensional code review: tiered reports by dimension + severity, agreement points marked with highest confidence, never rewrites code unless asked |
| `codemap` | Generate annotated repository structure maps for quick orientation, saving exploration tokens |
| `gh-cli` | GitHub CLI v2.97+ reference: pagination, repo targeting, discussions/projects/rulesets/skills, rate limit, gh-aw agentic CI, gh api fallback |
| `git-master` | Advanced Git operations: rebase, squash, fixup, bisect, reflog, code archaeology, worktree |
| `git-release` | Tag releases: release notes, SemVer inference, gh release commands |
| `resolving-merge-conflicts` | Resolve merge conflicts per-hunk: trace original intent, never invent new behavior, never --abort |
| `handoff` | Compress sessions into handoff documents (path references, no content duplication) |
| `opencode-config` | Write and maintain this repository's OpenCode config (agents/skills/commands/permissions) |
| `reflect` | Continuous improvement: identify friction → propose minimal maintainable fixes |
| `remove-deadcode` | Safely locate and delete dead code, verified via toolchain/LSP before deletion |
| `security-review` | Pre-merge security review (injection/XSS/SSRF/secrets/deserialization/path traversal), reports only, never fixes |
| `shared-language` | Build domain glossaries (CONTEXT.md), significantly saving tokens |
| `simplify` | Behavior-preserving code simplification (oracle analyzes → applies) |
| `spec-workflow` | Lightweight spec-driven changes: proposal → specs → design → tasks → archive |
| `verification-planning` | Plan the narrowest verification path before implementation |
| `verify-with-docs` | Verify API docs before coding, retrieval-first, prevents hallucination |
| `grilling` | Requirements alignment interview: one question at a time, multiple choice preferred, act after ambiguity converges |
| `tech-debt-audit` | 9-dimension tech debt audit (dead code/duplication/naming drift/complexity/dependencies/error handling/tests/docs/security), read-only report, no code changes |

## Design Decisions & Iteration History

Core concepts draw inspiration from [oh-my-openagent](https://github.com/code-yeongyu/oh-my-openagent) (intent gate, read-only isolation, anti-patterns), [oh-my-opencode-slim](https://github.com/alvinunreal/oh-my-opencode-slim) (scheduler priority, fallback chain, rejection contract, prompt cache safety, impact×confidence÷cost), [anomalyco/opencode](https://github.com/anomalyco/opencode) (configuration Schema, skill system), [cli/cli](https://github.com/cli/cli) (gh v2.97 command set, rate limit, gh-aw), [OpenSpec](https://github.com/Fission-AI/OpenSpec) (delta specs, OPSX action flow update/verify/four questions), [mattpocock/skills](https://github.com/mattpocock/skills) (conflict resolution discipline, handoff docs), [pi](https://github.com/earendil-works/pi) (answer first, edit later, concise responses, independent session collection), and [deepreview](https://github.com/mechanai/deepreview) (novelty classification convergence, effective-size routing, Points of Agreement). All implemented purely via configuration with zero extra dependencies.

> **Adapt, don't copy**: Heavy pipelines only contribute lightweight design concepts; redundant features are covered by existing agents/skills to avoid bloat. Adheres to the "simplify over add" principle, with each iteration aiming for a net reduction in tokens.
>
> **This round (v27) mechanism sources**: OPSX action flow (update/verify/four questions) internalized into spec-workflow; independent session context collection and prompt cache safety (stable static prefix, volatile content at payload tail) borrowed from pi and oh-my-opencode-slim; impact×confidence÷cost iteration gating enters deep-worker; Points of Agreement (highest-confidence annotation of agreement points) borrowed from deepreview; gh-cli adds rate limit and gh-aw from cli/cli v2.97.
>
> **Evaluated but not adopted**: mattpocock/skills' progressive disclosure and wait-what (existing skill lazy-loading already covers their value); superpowers has no config knobs, kept as plugin string injection.

### Iteration Milestones

27 iterations since v1, continuously aligning with upstream best practices:

- **v1-v7 (Foundation)**: Dual-model binding, agent role system, intent-gate routing, AGENTS.md global rules, skills directory, permission baseline
- **v8-v15 (Review + Specs + Contracts)**: code-review dual-axis calibration, spec-workflow, gh-cli alignment, rejection contract, background checks
- **v16-v22 (Continuous Slimming)**: Commands 29→18 (-38%), AGENTS.md 290→211 (-27%), no-op sentence trimming, schema validation dead-key removal
- **v23-v25 (Alignment + Security)**: Integrated 6 upstream repos, gh-cli v2.97 escape-injection advisory, procedure-driven prompt refinement, DCP window tuning
- **v26 (This Round's Slimming)**: prune:true and tool_output 800/20480 tightening, DCP switched to 60%/30% percentage thresholds, grilling introduced replacing writing-great-skills, opencode-config slimmed 131→64, code-review tiering + validator, gh-cli adds gh status, AGENTS.md adds User Override, orchestrator delegation cost discipline, 7 agent files net −22 lines
- **v27 (Deletion/Migration/Addition)**: Removed batch_tool dead config, ineffective `write: deny` on read-only agents, and 3 redundant bash rules; Context Management section migrated into an orchestrator-specific subsection; read-only agent bash whitelist, read adds `.env`; new tech-debt-audit skill; 15 skill descriptions slimmed 30-40%; gh-cli adds 5 points (rate limit/gh skill hosting/gh-aw etc.), code-review adds Points of Agreement, spec-workflow adds two update questions, orchestrator adds independent session collection + prompt cache safety, deep-worker adds impact×confidence÷cost

## Repository Structure

```text
├── opencode/                     # OpenCode config directory (deployable independently)
│   ├── .ai/
│   │   └── calibration.yml       # code-review severity calibration
│   ├── agents/                   # 10 specialized Agents
│   │   ├── orchestrator.md       # Main entry: intent gate + model-aware routing
│   │   ├── planner.md            # pro: architecture & planning
│   │   ├── deep-worker.md        # pro: heavy implementation
│   │   ├── oracle.md             # pro: deep code analysis (read-only)
│   │   ├── reviewer.md           # pro: dual-axis code review (read-only)
│   │   ├── consultant.md         # pro: solution discussion & advice
│   │   ├── ui-builder.md         # pro: frontend & UI
│   │   ├── explore.md            # flash: codebase search (read-only)
│   │   ├── librarian.md          # flash: external retrieval (read-only)
│   │   └── light-orchestrator.md # flash: simple editing
│   ├── skills/                   # 18 on-demand skills
│   │   ├── code-review/          # dual-axis parallel review + severity calibration
│   │   ├── codemap/              # generates repository structure map
│   │   ├── gh-cli/               # GitHub CLI v2.97+ reference + security advisory
│   │   ├── git-master/           # advanced Git operations
│   │   ├── git-release/          # Tag releases
│   │   ├── handoff/              # compress sessions into handoff docs
│   │   ├── opencode-config/      # meta-skill: this repo's config writing
│   │   ├── reflect/              # continuous improvement
│   │   ├── remove-deadcode/      # dead code detection & removal
│   │   ├── resolving-merge-conflicts/ # per-hunk conflict resolution discipline
│   │   ├── security-review/      # security review checklist
│   │   ├── shared-language/      # domain glossary (saves tokens)
│   │   ├── simplify/             # behavior-preserving code simplification
│   │   ├── spec-workflow/        # spec-driven development
│   │   ├── tech-debt-audit/      # tech debt audit (9 dimensions, read-only report)
│   │   ├── verification-planning/ # pre-implementation verification planning
│   │   ├── verify-with-docs/     # retrieval-first API verification
│   │   └── grilling/             # requirements alignment interview
│   ├── opencode.jsonc            # main config (18 commands)
│   ├── AGENTS.md                 # global rules
│   └── dcp.jsonc                 # DCP context compression (DeepSeek 128K, 60%/30% percentage thresholds)
├── README.md
├── LICENSE
└── README.*.md                   # READMEs in other languages
```

## Usage Guide

### Mode 1: Orchestrator Automatic Routing (Default)

Describe your requirements in natural language. The Orchestrator will automatically analyze intent, select the most suitable Agent and model, and execute.

```text
"Help me debug the login API error"     → oracle analyzes root cause → returns diagnostic report
"Optimize this loop, performance is poor" → oracle analyzes → deep-worker implements optimization
"Review this PR for me"                 → reviewer performs multi-dimensional review → returns tiered report
"I want to add an export feature to the user module" → planner drafts plan → deep-worker implements
"How to use React 19's use() API"       → librarian checks docs → returns signature and examples
```

### Mode 2: Direct via Command Aliases

| Scenario | Command |
| --- | --- |
| Complex implementations / multi-file changes | `/deep` |
| Lightweight modifications / single-file edits | `/quick` |
| Draft technical solutions / architecture design | `/plan` |
| Debugging / deep analysis | `/oracle` |
| Code review | `/review` |
| External search / API lookup | `/search` |
| Frontend / UI work | `/ui` |
| Solution discussion / trade-off analysis | `/consult` |
| Structured debugging | `/oracle` |

### Typical Workflows

**Develop new features (spec-driven):**
```text
/spec-propose  → /spec-apply  → /review
```

**Debugging:**
```text
/oracle  → /deep  → /rmslop  → /commit
```

**Code review:**
```text
/review-pr   ← review PR + auto-reply on GitHub
/review      ← dual-axis parallel review
```

## Design Philosophy

- **Purely config-driven, zero extra dependencies** — All capabilities are implemented via `opencode.jsonc` + `agents/*.md` + `skills/*/SKILL.md` + `AGENTS.md`
- **Maximized use of DeepSeek V4 dual-models** — Pro handles reasoning & decision-making, Flash handles queries & lightweight execution
- **Token efficiency first** — Path references instead of pasting files, on-demand skill loading, tiered compression management
- **Plugins enhance without overpowering** — superpowers provides process discipline, DCP provides intelligent compression instead of blunt truncation (adaptive percentage thresholds, native compaction as fallback)
- **Separation of execution and exploration** — deep-worker/light-orchestrator prohibit research/delegation, explore/librarian prohibit modifications
- **Continuous improvement** — reflect mechanizes friction detection, dual-axis calibration in code-review ensures quality
