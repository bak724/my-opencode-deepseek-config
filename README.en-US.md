

# My OpenCode × DeepSeek Config

**OpenCode × DeepSeek Optimal Config** —— A configuration scheme designed to maximize the capabilities of the DeepSeek V4 dual-model (Pro + Flash) within the OpenCode multi-agent framework. Core philosophy: **Token efficiency first, achieving the best development results with the minimum context cost**.

## Current Configuration Overview

- Default main Agent: `orchestrator`
- Main model: `deepseek/deepseek-v4-pro`, lightweight model: `deepseek/deepseek-v4-flash`
- Agent depth: `subagent_depth: 3` (supports 3 levels of agent nesting)
- Model isolation: `enabled_providers: ["deepseek"]` + `disabled_providers` double lock
- Session sharing: disabled (`share: "disabled"`); snapshots: enabled (`snapshot: true`)
- Permission baseline: allow by default, destructive bash commands set to `ask`; `.env`-like sensitive files set to `deny`; external directories set to `ask`
- Context compression: DCP proactive compression (35K-75K threshold) + OpenCode native compaction fallback
- Global rules: `AGENTS.md` (core principles, task rejection contract, context & token efficiency, self-verification, anti-patterns, etc.)
- Skills: **17** `SKILL.md` files in the `skills/` directory, loaded on-demand via the native `skill` tool
- Plugins: `superpowers` (14 process-type skills), `@tarquinen/opencode-dcp` (intelligent context trimming)
- Experimental features: `batch_tool` is enabled by default

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

### Method 1: Clone to Global Config Directory (Recommended)

**Windows (PowerShell):**

```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.config"
git clone https://github.com/znlgis/my-opencode-deepseek-config.git "$env:USERPROFILE\.config\opencode"
```

**Linux / macOS:**

```bash
git clone https://github.com/znlgis/my-opencode-deepseek-config.git ~/.config/opencode
```

> **Compatibility Note**: `~/.config/opencode` is the standard global config path for OpenCode. The files in this repository (e.g., `agents/`, `skills/`, `AGENTS.md`) follow OpenCode's convention layout and will be automatically recognized upon cloning without additional configuration.

### Method 2: Clone to Any Location + Environment Variables

```powershell
$env:OPENCODE_CONFIG_DIR = "D:\path\to\opencode-config"
opencode
```

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
| `code-review` | Dual-axis parallel review (specifications + conventions) + severity calibration |
| `codemap` | Generates annotated repository structure maps to save exploration tokens |
| `gh-cli` | Comprehensive reference for GitHub CLI v2.96+ (Issues 2.0, copilot, agent-task) |
| `gh-skill` | Discover, install, update, and publish Agent skills |
| `git-master` | Advanced Git operations: rebase, squash, bisect, reflog, worktree |
| `git-release` | Tag releases: SemVer inference, release notes, gh release commands |
| `handoff` | Compress sessions into handoff documents (path references, no content duplication) |
| `opencode-config` | Write and maintain OpenCode configurations |
| `reflect` | Continuous improvement: identify friction → propose minimal fixes |
| `remove-deadcode` | Safely locate and delete dead code with LSP verification prior to deletion |
| `security-review` | Pre-merge security review for diffs |
| `shared-language` | Build domain glossaries, significantly saving context tokens |
| `simplify` | Behavior-preserving code simplification (oracle analyzes → light-orchestrator applies) |
| `spec-workflow` | Lightweight spec-driven changes (propose → design → tasks → implement → archive) |
| `verification-planning` | Plan the narrowest verification path before implementation |
| `verify-with-docs` | Verify APIs against documentation before coding, prioritize retrieval to prevent hallucinations |
| `writing-great-skills` | Skill writing standards: no-op trimming, positive phrasing, completion criteria |

## Design Decisions & Iteration History

Core concepts draw inspiration from [oh-my-openagent](https://github.com/code-yeongyu/oh-my-openagent) (intent gate, read-only isolation, anti-patterns), [oh-my-opencode-slim](https://github.com/alvinunreal/oh-my-opencode-slim) (scheduler priority, fallback chain, rejection contract), [anomalyco/opencode](https://github.com/anomalyco/opencode) (configuration Schema, skill system), [cli/cli](https://github.com/cli/cli) (full gh command set), [OpenSpec](https://github.com/Fission-AI/OpenSpec) (delta specs, change proposal updates), [mattpocock/skills](https://github.com/mattpocock/skills) (handoff docs, structured debugging), [pi](https://github.com/earendil-works/pi) (answer first, modify later, concise responses), and [deepreview](https://github.com/mechanai/deepreview) (entropy scanning, convergence checks). All implemented purely via configuration with zero extra dependencies.

> **Adapt, don't copy**: Heavy pipelines only contribute lightweight design concepts; redundant features are covered by existing agents/skills to avoid bloat. Adheres to the "refactor over add" principle, with each iteration aiming for a net reduction in tokens.

### Iteration Milestones

| Phase | Key Changes |
| --- | --- |
| **v1-v7 (Foundation)** | Dual-model binding, agent role system, intent gate/classification routing, global AGENTS.md rules, skills directory & command aliases, permission baseline |
| **v8-v12 (Review + Specs)** | Enhanced code-review (tiering/self-check/rejection criteria), established spec-workflow (explore→propose→apply→archive), added deepwork/reflect/verification-planning, gh-cli aligned to v2.96+ |
| **v13-v15 (Contracts + Streamlining)** | Added Evidence Discipline / Task Rejection Contract / stop condition to AGENTS.md; fully deduplicated agent prompts & global rules; completed background sub-agent error checks |
| **v16-v18 (Efficient Execution)** | Removed mythological names, merged routing tables, expanded gh-cli to Issues 2.0, added verify + decision framework to spec-workflow |
| **v19 (Upstream Alignment)** | Reviewed 6 upstream repos; fixed `/review-pr` inline comment bug; changed code-review routing from raw line count to effective logical volume |
| **v20 (Refactoring Optimization)** | `agent/`→`agents/` aligned with OpenCode recommendations; AGENTS.md streamlined by 22% (292→229 lines); added `diagnose` (6-stage debugging) + `handoff` (session handoff) skills; spec-workflow added `/update`; code-review added entropy scanning + convergence checks; agent prompts deduplicated by 20% |
| **v21 (Comprehensive Slimming)** | Skills reduced from 18→17 (removed deepwork/conventional-commits/diagnose, added writing-great-skills/shared-language); commands reduced from 29→18 (-38%); AGENTS.md reduced from 227→212 lines (-7%); sentence-by-sentence no-op trimming on skills. Dual-axis parallel code-review + calibration file mechanism. Leveraged practical experience from 6 repos like pi/deepreview/mattpocock. |
| **v22 (Schema Validation Slimming)** | Verified against official OpenCode & DCP schemas: removed invalid `agent.fallback` dead key; confirmed all `dcp.jsonc` keys are valid (v3.1.14), zero blind additions; merged 'Token Efficiency' into 'Context Management' in AGENTS.md, fully deduplicated, fixed dangling `Self-Verification` reference (212→197 lines); merged orchestrator's three routing tables (128→80 lines, -37%, Intent Gate/Agent Directory/Fallback all preserved); stripped ignored `license/compatibility/metadata` frontmatter from 14 skills (-70 lines); downgraded `tool_output` to proactive token savings (1500 lines/40KB). |

## Repository Structure

```text
├── .ai/
│   └── calibration.yml           # code-review severity calibration
├── agents/                       # 10 specialized Agents
│   ├── orchestrator.md           # Main entry: intent gate + model-aware routing
│   ├── planner.md                # pro: architecture & planning
│   ├── deep-worker.md            # pro: heavy implementation
│   ├── oracle.md                 # pro: deep code analysis (read-only)
│   ├── reviewer.md               # pro: dual-axis code review (read-only)
│   ├── consultant.md             # pro: solution discussion & advice
│   ├── ui-builder.md             # pro: frontend & UI
│   ├── explore.md                # flash: codebase search (read-only)
│   ├── librarian.md              # flash: external retrieval (read-only)
│   └── light-orchestrator.md     # flash: simple editing
├── skills/                       # 17 on-demand skills
│   ├── code-review/              # dual-axis parallel review + severity calibration
│   ├── codemap/                  # generates repository structure map
│   ├── gh-cli/                   # GitHub CLI v2.96+ reference
│   ├── gh-skill/                 # gh skill management
│   ├── git-master/               # advanced Git operations
│   ├── git-release/              # Tag releases
│   ├── handoff/                  # compress sessions into handoff docs
│   ├── opencode-config/          # meta-skill: config writing
│   ├── reflect/                  # continuous improvement
│   ├── remove-deadcode/          # dead code detection & removal
│   ├── security-review/          # security review checklist
│   ├── shared-language/          # domain glossary (saves tokens)
│   ├── simplify/                 # behavior-preserving code simplification
│   ├── spec-workflow/            # spec-driven development
│   ├── verification-planning/    # pre-implementation verification planning
│   ├── verify-with-docs/         # retrieval-first API verification
│   └── writing-great-skills/     # skill writing standards
├── opencode.jsonc                # main config (18 commands)
├── AGENTS.md                     # global rules (~197 lines)
├── dcp.jsonc                     # DCP context compression (DeepSeek 128K, schema v3.1.14 verified)
├── LICENSE
└── README.md
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

- **Purely config-driven, zero extra dependencies** —— — All capabilities are implemented via `opencode.jsonc` + `agents/*.md` + `skills/*/SKILL.md` + `AGENTS.md`
- **Maximized use of DeepSeek V4 dual-models** / — Pro handles reasoning & decision-making, Flash handles queries & lightweight execution
- **Token efficiency first** / — Path references instead of pasting files, on-demand skill loading, tiered compression management
- **Plugins enhance without overpowering** / — superpowers provides process discipline, DCP provides intelligent compression instead of blunt truncation
- **Separation of execution and exploration** / — deep-worker/light-orchestrator prohibit research/delegation, explore/librarian prohibit modifications
- **Continuous improvement** / — reflect mechanizes friction detection, dual-axis calibration in code-review ensures quality
