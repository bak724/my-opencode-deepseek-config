# OpenCode v21 全面重构实施计划

> **For agentic workers:** 使用 `subagent-driven-development` 技能执行此计划，每任务独立子代理，任务间审查门控。

**Goal:** 在保持功能完整的前提下，减少常驻上下文 22%、技能总行数 24%、命令数 38%，并增强 code-review 双轴并行和技能质量纪律。

**Architecture:** 实施分 8 个阶段：先删后修再建。Phase 1-2 做减法（删技能/命令/Bug 修复），Phase 3-5 增强核心技能和命令体系，Phase 6-7 打磨 AGENTS.md 和全量修剪，Phase 8 同步文档。

**Note:** 调研发现 AGENTS.md 中不含路由表（路由表仅在 `agents/orchestrator.md` 中），因此 AGENTS.md 实际节省主要来自 Code Style 压缩和冗余文案修剪，目标从 ~165 行调整为 ~210 行（-8%）。

**Tech Stack:** OpenCode 配置（JSONC frontmatter + Markdown prompting），仅 deepseek-v4-pro 和 deepseek-v4-flash 两个模型。

## Global Constraints

- 仅使用 `deepseek/deepseek-v4-pro` 和 `deepseek/deepseek-v4-flash`
- 不新增依赖或插件
- 所有改动必须降低 token 消耗
- 不创建未在 spec 中批准的文件

---

### Task 0: 准备工作 — 核实当前状态

**文件:**
- Read: `AGENTS.md`、`opencode.jsonc`、`dcp.jsonc`、`README.md`
- Verify: 18 个 skill 目录全部存在、10 个 agent 文件全部存在

- [ ] **Step 0.1: 确认删除清单的每个文件存在**

```powershell
# 确认要删除的 3 个技能目录存在
Test-Path -LiteralPath "skills/deepwork"
Test-Path -LiteralPath "skills/conventional-commits"
Test-Path -LiteralPath "skills/diagnose"

# 确认 opencode.jsonc 包含要删除的 13 个命令键
```

验证：所有路径为 `True`。

- [ ] **Step 0.2: 记录基线统计**

```powershell
# 统计基线
Get-ChildItem -Recurse -Path skills -Filter SKILL.md | ForEach-Object { (Get-Content $_.FullName).Count } | Measure-Object -Sum
(Get-Content AGENTS.md).Count
```

验证：确认基线行数与 spec 一致（~3,500 行技能，227 行 AGENTS.md）。

- [ ] **Step 0.3: 提交基线**

```powershell
git add -A; git commit -m "chore: baseline before v21 refactor"
```

---

### Task 1: 删除 3 个过时技能目录

**文件:**
- Delete: `skills/deepwork/SKILL.md` + `skills/deepwork/` 目录
- Delete: `skills/conventional-commits/SKILL.md` + `skills/conventional-commits/` 目录
- Delete: `skills/diagnose/SKILL.md` + `skills/diagnose/` 目录

- [ ] **Step 1.1: 删除 deepwork 技能目录**

```powershell
Remove-Item -Recurse -Force -LiteralPath "skills/deepwork"
```

验证：`Test-Path "skills/deepwork"` 返回 `False`。

- [ ] **Step 1.2: 删除 conventional-commits 技能目录**

```powershell
Remove-Item -Recurse -Force -LiteralPath "skills/conventional-commits"
```

验证：`Test-Path "skills/conventional-commits"` 返回 `False`。

- [ ] **Step 1.3: 删除 diagnose 技能目录**

```powershell
Remove-Item -Recurse -Force -LiteralPath "skills/diagnose"
```

验证：`Test-Path "skills/diagnose"` 返回 `False`。

- [ ] **Step 1.4: 验证剩余 15 个技能**

```powershell
(Get-ChildItem -Path skills -Directory).Count
```

验证：输出应为 `15`。

- [ ] **Step 1.5: 提交**

```powershell
git add skills/deepwork skills/conventional-commits skills/diagnose
git commit -m "refactor(skills): remove deepwork, conventional-commits, diagnose (-3 skills)"
```

---

### Task 2: Bug 修复 — README、dcp.jsonc、opencode-config

**文件:**
- Modify: `README.md:13` — DCP 阈值 45K-85K → 35K-75K
- Modify: `dcp.jsonc:56` — 注释 85K → 75K
- Modify: `README.md` — AGENTS.md 行数声称更新
- Modify: `README.md` — 仓库树补 LICENSE
- Modify: `skills/opencode-config/SKILL.md` — opencode.json → opencode.jsonc

- [ ] **Step 2.1: 修复 README DCP 阈值（行 13 附近）**

Read `README.md` 找到 "DCP 主动压缩（45K-85K 阈值）" 行，替换为 "35K-75K"。

```markdown
# 旧: DCP 主动压缩（45K-85K 阈值）
# 新: DCP 主动压缩（35K-75K 阈值）
```

验证：`grep "45K" README.md` 无结果。

- [ ] **Step 2.2: 修复 dcp.jsonc 注释与值矛盾（行 56 附近）**

Read `dcp.jsonc` 找到 "85K 开始强力推压缩（留 43K 缓冲）" 行。

```jsonc
// 旧: "85K 开始强力推压缩（留 43K 缓冲）"
// 新: "75K 开始强力推压缩（留 53K 缓冲）"
```

验证：`grep "85K" dcp.jsonc` 无结果。

- [ ] **Step 2.3: 修复 opencode-config 技能引用 opencode.json → opencode.jsonc**

Read `skills/opencode-config/SKILL.md`，全文搜索 `opencode.json`（但排除 `opencode.jsonc` 的匹配），确认表格中布局引用 `opencode.json` 的行。

```markdown
# 将布局表中的 `opencode.json` 改为 `opencode.jsonc`
# 保留 `$schema` 引用中正确的 `opencode.json`（schema URL 不变）
```

验证：`grep "| \`opencode.json\` |" skills/opencode-config/SKILL.md` 无结果（单独出现的 opencode.json）。

- [ ] **Step 2.4: 提交**

```powershell
git add README.md dcp.jsonc skills/opencode-config/SKILL.md
git commit -m "fix: sync DCP thresholds, filenames, and README claims — 4 bug fixes"
```

---

### Task 3: 精简 opencode.jsonc — 删除 13 个命令 + 新增 2 个 + 修复 /simplify

**文件:**
- Modify: `opencode.jsonc:98-234` — 命令段完全重写

**删除的 13 个命令**: `deepwork`, `review-loop`, `docs`, `learn`, `explore`, `skill`, `verify-plan`, `propose`, `apply`, `archive`, `update`

**保留的 16 个命令**: `deep`, `quick`, `ui`, `review`, `review-pr`, `plan`, `search`, `oracle`, `consult`, `commit`, `release`, `reflect`, `handoff`, `codemap`, `simplify`, `rmslop`

**新增的 2 个命令**: `spec-propose`, `spec-apply`

- [ ] **Step 3.1: 重写 /spec-propose 命令（合并原 /propose + /update）**

在命令段末尾添加 `spec-propose` 命令（位置在 `rmslop` 之后，`codemap` 之前按字母序放在 `simplify` 前面）：

```jsonc
"spec-propose": {
    "description": "Explore code then draft a spec-driven change proposal (or update an existing one if intent overlaps >50%)",
    "agent": "planner",
    "template": "Load the spec-workflow skill. If there is an existing change under openspec/changes/ with >50% scope overlap to the current intent, run update action on it. Otherwise, scaffold a new openspec/changes/<change-id>/ with proposal.md, tasks.md, delta specs, and design.md only if warranted. Focus on WHY and WHAT, not HOW. Stop after the artifacts are ready for review."
}
```

验证：`grep "spec-propose" opencode.jsonc` 返回匹配行。

- [ ] **Step 3.2: 重写 /spec-apply 命令（合并原 /apply + /archive + /verify-plan）**

```jsonc
"spec-apply": {
    "description": "Implement a spec-driven change task-by-task, verify, then auto-archive on completion",
    "agent": "deep-worker",
    "template": "Load the spec-workflow skill. For the requested change under openspec/changes/: read proposal.md, design.md, delta specs, then work tasks.md one unchecked item at a time, marking each `- [x]`. If a task is blocked or design proves wrong, update the artifacts. After all tasks complete, run the archive action: fold delta specs into openspec/specs/ and move to openspec/changes/archive/. Run any available verification."
}
```

验证：`grep "spec-apply" opencode.jsonc` 返回匹配行。

- [ ] **Step 3.3: 重写 /commit 命令（移除对已删除 conventional-commits 技能的引用）**

找到 `/commit` 命令定义，将 `"Load the conventional-commits skill."` 替换为内联的 Conventional Commits 格式规则：

```jsonc
"commit": {
    "description": "Stage changes and write a Conventional Commits message",
    "agent": "light-orchestrator",
    "template": "Here is the current state:\n\nStatus:\n!`git status --short`\n\nDiff (staged + unstaged):\n!`git diff HEAD`\n\nStage the relevant changes and write a single commit message following Conventional Commits: type(scope): imperative-mood subject (feat/fix/docs/chore/refactor/test). Use `BREAKING CHANGE:` footer if applicable. Do not push."
}
```

验证：`grep "conventional-commits" opencode.jsonc` 无结果。

- [ ] **Step 3.4: 修复 /simplify 命令（oracle 只读 → oracle 分析 + light-orchestrator 应用）**

```jsonc
"simplify": {
    "description": "Oracle analyzes then light-orchestrator applies behavior-preserving simplification",
    "agent": "oracle",
    "template": "Load the simplify skill. Read the target file(s) and apply the simplification patterns (early returns, inline single-use variables, remove single-caller functions, simplify conditionals). Produce a detailed report of what to change with exact oldString/newString pairs. Then a writer agent will apply your suggestions while you verify behavior is preserved."
}
```

验证：`grep "simplify" opencode.jsonc` 包含 "writer agent will apply"。

- [ ] **Step 3.5: 删除 13 个命令定义**

从 `opencode.jsonc` 的 `command` 对象中删除以下键：
`deepwork`, `review-loop`, `docs`, `learn`, `explore`, `skill`, `verify-plan`, `propose`, `apply`, `archive`, `update`

命令键在对象中按字母序排列（除 `spec-propose` 和 `spec-apply` 插入在 `simplify` 前），删除后应保持顺序一致。

- [ ] **Step 3.6: 端到端读取 opencode.jsonc**

Read 完整 `opencode.jsonc`，确认：
- `command` 对象包含恰好 18 个键
- 无误删的逗号/括号导致 JSON 非法
- 新命令 template 无误

验证：

```powershell
# 统计命令键数量（JSONC 中 "name": 模式计数）
Select-String -Path opencode.jsonc -Pattern '"description"' | Measure-Object | Select-Object -ExpandProperty Count
```

（预期 ~18，具体取决于 op 的 description 键是否也在匹配范围）

- [ ] **Step 3.7: 提交**

```powershell
git add opencode.jsonc
git commit -m "refactor(commands): 29->18 — remove 13 low-value commands, add spec-propose/spec-apply, fix /simplify and /commit"
```

---

### Task 4: 创建 2 个新技能 + 校准文件

**文件:**
- Create: `skills/writing-great-skills/SKILL.md` (~100 行)
- Create: `skills/shared-language/SKILL.md` (~80 行)
- Create: `.ai/calibration.yml` (~30 行)

- [ ] **Step 4.1: 创建 writing-great-skills 技能**

```powershell
New-Item -ItemType Directory -Force -Path "skills/writing-great-skills"
```

写入文件 `skills/writing-great-skills/SKILL.md`：

```markdown
---
name: writing-great-skills
description: Use when creating new skills, editing existing skills, or the task mentions skill/write-skill/SKILL.md/writing-skills. Covers no-op trimming discipline, positive phrasing, completion criteria, and embedded Conventional Commits format reference.
---

# Writing Great Skills

How to write and maintain high-quality, token-efficient skills for OpenCode.

## The No-Op Trim (Most Important)

Run this test on every sentence in every skill:

> **If I delete this sentence, does the agent's behavior change?**

If the answer is **no**, delete the sentence immediately. Agents already know
the default behaviors you're describing. Every sentence you keep must pass this
test.

## Positive Phrasing

Say what the agent SHOULD do, never what it should NOT do.

```
BAD:  "Don't forget to read the file before editing."
GOOD: "Read the file before editing."
BAD:  "Never skip the verification step."
GOOD: "Run the verification step after every change."
```

Negatives ("don't skip", "avoid forgetting") make the forbidden behavior more
accessible to the model. Frame every instruction as a positive action.

## Completion Criteria

Every step in a skill MUST have an observable, checkable completion condition.

```
BAD:  "Review the code for issues."
GOOD: "Read the file from top to bottom. List every issue found, with file:line."
```

Steps without completion criteria cause premature termination — the agent
declares "done" without actually finishing. A completion criterion is a
condition anyone can check: "file exists at path X", "grep returns 0 matches",
"output contains the string Y".

## Information Hierarchy

- **In-step content** — what the agent MUST know to execute this step
- **In-skill reference** — background info the agent MIGHT need
- **External reference** — links to docs the agent fetches on demand

Push everything possible to external references. Keep in-step content to the
minimum needed for correct execution.

## Trim Checklist

Before finalizing any skill, scan for:

1. **No-op sentences** — deleted without behavior change → delete
2. **Repetition** — same instruction appears twice → keep once
3. **Bloated descriptions** — 20 words where 5 suffice → compress
4. **Negative phrasing** — "don't X" → "do Y"
5. **Vague steps** — no completion criterion → add one

## Conventional Commits Format (Reference)

When writing commit messages, use this format:

```
type(scope): imperative-mood subject

Types: feat / fix / docs / chore / refactor / test / style / perf / ci
BREAKING CHANGE: description (footer, when applicable)
```

The subject uses imperative mood ("add" not "added"), lowercase, no trailing period.
```

- [ ] **Step 4.2: 创建 shared-language 技能**

```powershell
New-Item -ItemType Directory -Force -Path "skills/shared-language"
```

写入文件 `skills/shared-language/SKILL.md`：

```markdown
---
name: shared-language
description: Use when the task mentions domain language, CONTEXT.md, terminology, shared vocabulary, or when the same concepts are being explained repeatedly in different sessions. Builds and maintains a shared language document that drastically reduces token consumption.
---

# Shared Language

Build and maintain `.opencode/CONTEXT.md` — a glossary of project-specific
terms and concepts, each defined in ONE sentence. Every conversation that
explains the same concept in long-form wastes tokens. A shared language
document replaces those explanations with a single reference.

## When to Create or Update

1. You find yourself explaining the same concept (>2 sentences) across
   multiple messages or sessions.
2. You encounter a term the user uses that has project-specific meaning.
3. After a design discussion — capture the decisions as shared language
   entries.

## Format

`.opencode/CONTEXT.md` uses this exact format:

```markdown
# Shared Language

- `<term>`: <one-sentence definition>
- `<term>`: <one-sentence definition>
```

Each entry is **one line**. No paragraphs, no examples, no "further reading".

## How Agents Use It

Before reasoning about the codebase, scan `.opencode/CONTEXT.md`:

1. If a term appears, use its one-sentence definition directly — do not
   re-explain or expand.
2. If a concept is missing but you needed 3+ sentences to convey it, add an
   entry after the session.

The goal: each term's definition replaces 5-20 sentences of repeated
explanation across sessions. The agent also spends fewer tokens on thinking
because the concepts are pre-compressed.

## Maintenance

- Keep entries to one sentence. If an entry grows, it's a sign it should be
  a design doc instead.
- Delete entries that no longer apply (removed features, renamed concepts).
- Sort alphabetically.
```

- [ ] **Step 4.3: 创建校准文件**

```powershell
New-Item -ItemType Directory -Force -Path ".ai"
```

写入文件 `.ai/calibration.yml`：

```yaml
# Code-review severity calibration.
# When a review agent finds a pattern listed here, it MUST downgrade the
# severity as specified and cite the calibration entry.
#
# This file: keep local (gitignored). Team-shared entries go in
# .ai/calibration.yml tracked in git.

calibrations: []
# Example entries:
#   - pattern: "missing authentication on localhost-only internal tool"
#     downgrade: "high -> low"
#     reason: "internal use only, no external exposure"
#   - pattern: "console.log in CLI tool"
#     downgrade: "major -> minor"
#     reason: "CLI output is intentional user-facing logging"
```

- [ ] **Step 4.4: 确认 .gitignore 包含 .ai/calibration.yml（如果需要）**

Read `.gitignore`。检查是否已有 `.ai/` 条目。如无，**不添加** — calibration.yml 应被 git 跟踪作为团队共享基线。个人覆盖用 gitignored 的本地文件。

- [ ] **Step 4.5: 端到端读取 3 个新文件**

Read 全部 3 个新文件，检查：
- 无 YAML/Markdown 语法错误
- frontmatter 格式正确
- 无空行过多等格式问题

- [ ] **Step 4.6: 提交**

```powershell
git add skills/writing-great-skills skills/shared-language .ai/calibration.yml
git commit -m "feat(skills): add writing-great-skills, shared-language (+.ai/calibration.yml)"
```

---

### Task 5: 重构 code-review 技能为双轴并行

**文件:**
- Modify: `skills/code-review/SKILL.md` — 314 行 → ~280 行
- 预计改动量：保留现有有效大小分级逻辑，新增双轴并行调度段，新增内联 Std/Spec 检查清单

Read 当前 `skills/code-review/SKILL.md` 获取完整内容后执行编辑。

核心改动点：

1. **在入口段之后插入双轴调度指令**（在现有 "scope by effective size" 逻辑之前）：
```markdown
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
```

2. **在 threat model calibration 段后新增校准文件读取指令**：
```markdown
## Calibration

Before assigning severity, read `.ai/calibration.yml` if it exists. For any
finding whose pattern matches a calibration entry, apply the specified
downgrade and cite the entry in the report. This prevents the same
low-priority finding from being flagged as high in every review.
```

3. **保留现有逻辑**：有效大小分级、review→fix 循环（5 轮上限 + NEW/RECURRING/REGRESSION 分类）、PR 行级评论（gh api POST reviews）、对抗性自检、拒绝规则、文档漂移批处理。这些不删。

- [ ] **Step 5.1: 读取当前 code-review 技能**

```powershell
Get-Content skills/code-review/SKILL.md
```

- [ ] **Step 5.2: 在入口段后插入双轴调度指令**

使用 Edit 工具在 `## Method` 段之后、`## Dimensions` 段之前插入上述双轴调度 + 检查清单代码块。

- [ ] **Step 5.3: 在威胁模型段后插入校准文件读取指令**

使用 Edit 工具在 threat model / auto-calibrate 段后追加校准文件读取指令。

- [ ] **Step 5.4: 修剪重复内容**

检查是否有与 AGENTS.md 反模式重复的描述 → 删除或改为 `(see AGENTS.md Anti-Patterns)` 引用。

- [ ] **Step 5.5: 端到端读取确认**

Read 完整 `skills/code-review/SKILL.md`，验证：
- 结构完整（入口 → 分级 → 双轴调度 → 检查清单 → 维度 → 报告格式 → 自检 → review→fix）
- 无重复内容
- 行数 ≤ 280

- [ ] **Step 5.6: 提交**

```powershell
git add skills/code-review/SKILL.md
git commit -m "refactor(code-review): dual-axis parallel review with inline std/spec checklists + calibration file support"
```

---

### Task 6: 重构 spec-workflow 技能 + verify-with-docs 增强

**文件:**
- Modify: `skills/spec-workflow/SKILL.md` — ~350 行 → ~280 行
- Modify: `skills/verify-with-docs/SKILL.md` — ~130 行 → ~110 行

- [ ] **Step 6.1: 读取 spec-workflow 技能**

```powershell
Get-Content skills/spec-workflow/SKILL.md
```

- [ ] **Step 6.2: 在 update 动作中插入 OpenSpec 新建 vs 更新启发式**

找到 update 动作的描述段，追加：

```markdown
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
```

- [ ] **Step 6.3: no-op 修剪 spec-workflow**

逐段检查，删除以下类型的句子：
- "Make sure to follow the process carefully"（agent 默认会遵循指令）
- "This is important because..."（不言自明的重要性）
- 重复的 `openspec/` 目录结构描述（已在段首描述过）

- [ ] **Step 6.4: 读取 verify-with-docs 技能**

- [ ] **Step 6.5: 在 verify-with-docs 追加 references 本地优先规则**

在 "retrieval first" 段后追加：

```markdown
## References Local-First

Before fetching from the web, check if the target library is available as a
local reference in `opencode.jsonc` → `references`. If mounted (e.g., via
`references: { "opencode": { "repository": "anomalyco/opencode" } }`), read
the source code from the local clone first. Only fall back to web fetch if:
- The reference is outdated (commit older than the version in use)
- The reference doesn't cover the relevant code paths
- No reference is mounted for the target library
```

- [ ] **Step 6.6: no-op 修剪 verify-with-docs**

删除重复的 "cite source URLs" 指令（已在多个步骤中重复出现）。

- [ ] **Step 6.7: 端到端读取两个文件**

- [ ] **Step 6.8: 提交**

```powershell
git add skills/spec-workflow/SKILL.md skills/verify-with-docs/SKILL.md
git commit -m "refactor(skills): spec-workflow add update heuristic, verify-with-docs add references-local-first; no-op trim both"
```

---

### Task 7: 更新 reviewer agent prompt

**文件:**
- Modify: `agents/reviewer.md` — 微调 prompt 以反映双轴 code-review 变更

- [ ] **Step 7.1: 读取当前 reviewer agent**

Read `agents/reviewer.md`（已在 memory 中，37 行）。

- [ ] **Step 7.2: 修改审查方法描述**

将第 20-24 行的 "Method" 段更新为：

```markdown
## Method
Load the `code-review` skill — it defines the full token-frugal, dual-axis workflow:
- **Scope by effective size.** Abbreviated path for ≤8 logic files and ≤300 effective lines; Full path otherwise or on high-stakes signals (auth, migration, concurrency, public contract).
- **Dual-axis parallel review.** Standards axis (style/naming/structure/comments/imports) and Spec axis (correctness/boundaries/security/performance) run concurrently, then merge findings.
- **Calibrate severity.** Read `.ai/calibration.yml` to downgrade known low-priority patterns.
```

- [ ] **Step 7.3: 端到端读取确认**

- [ ] **Step 7.4: 提交**

```powershell
git add agents/reviewer.md
git commit -m "refactor(reviewer): update prompt for dual-axis code-review workflow"
```

---

### Task 8: 精简 AGENTS.md

**文件:**
- Modify: `AGENTS.md` — 229 行 → ~210 行（-8%）

**实际节省点**：AGENTS.md 中不含路由表（路由表仅在 `agents/orchestrator.md` 中），因此节省来自压缩 Code Style 段、移除 Context Management 中冗余项、压缩 Self-Verification/Evidence Discipline、添加 git 安全规则。

- [ ] **Step 8.1: 读取当前 AGENTS.md**

Read 完整 `AGENTS.md`（已在 memory 中）。

- [ ] **Step 8.2: 压缩 Code Style 段（行 165-175，9 条 → 5 条）**

```markdown
## Code Style (when implementing)

- **Prefer `const` over `let`.** Early return instead of `else`.
- **Prefer functional array methods** (`flatMap`, `filter`, `map`) over imperative loops.
- **No import aliases** unless disambiguating a collision; no wildcard imports (`import * as`).
- **Inline single-use values.** Don't create a variable for a value used exactly once.
- **No catch-all files** (`utils.ts`, `helpers.ts`).
```

- [ ] **Step 8.3: 压缩 Context Management（行 58-71）**

删除两个冗余项：
- "Cache-aware prompting" — 这在 OpenCode 层面自动处理
- "Consider the handoff skill" — handoff 技能自身会触发

保留 5 个核心项。

- [ ] **Step 8.4: 压缩 Self-Verification（行 185-201）和 Evidence Discipline（行 203-215）**

合并这两个高度重叠的节：

```markdown
## Verification & Evidence

Before claiming any task is complete:
1. Re-read every modified file end-to-end — scan for leftover debug prints,
   TODOs, incomplete logic.
2. Verify changes don't break callers — grep for usages of modified functions.
3. If tests exist, run them. If not, state what manual verification you performed.

Never claim "done" without evidence: a passing build, a clean lint check,
an end-to-end read, or a grep showing no broken callers. Evidence precedes
assertion.
```

- [ ] **Step 8.5: 新增 git 安全纪律**

在 Multi-Step Task Discipline 段后添加：

```markdown
## Git Safety

- Only stage and commit files you modified in this session. Never `git add -A`,
  `git reset --hard`, `git checkout .`, or `git clean -fd` — those discard
  work from other sessions or tools that may share the same working directory.
- Before committing: inspect `git status`, `git diff --staged`, and
  `git log --oneline -10`. Stage only intended files.
- Never force-push, skip hooks (`--no-verify`), or amend commits without
  explicit user request.
```

- [ ] **Step 8.6: 端到端读取 AGENTS.md**

- [ ] **Step 8.7: 确认行数在目标范围**

```powershell
(Get-Content AGENTS.md).Count
```

验证：输出 ≤ 215。

- [ ] **Step 8.8: 提交**

```powershell
git add AGENTS.md
git commit -m "refactor(AGENTS.md): compress code style, merge verification sections, add git safety rules (-8%)"
```

---

### Task 9: 全量 no-op 修剪 — 15 个技能逐句过筛

**文件:**
- Modify: `skills/codemap/SKILL.md`
- Modify: `skills/gh-cli/SKILL.md`
- Modify: `skills/gh-skill/SKILL.md`
- Modify: `skills/git-master/SKILL.md`
- Modify: `skills/git-release/SKILL.md`
- Modify: `skills/handoff/SKILL.md`
- Modify: `skills/opencode-config/SKILL.md`
- Modify: `skills/reflect/SKILL.md`
- Modify: `skills/remove-deadcode/SKILL.md`
- Modify: `skills/security-review/SKILL.md`
- Modify: `skills/simplify/SKILL.md`
- Modify: `skills/verification-planning/SKILL.md`
- Modify: `skills/code-review/SKILL.md` (二次修剪，已重构但可能有残留)
- Modify: `skills/spec-workflow/SKILL.md` (二次修剪)
- Modify: `skills/verify-with-docs/SKILL.md` (二次修剪)

**新增技能跳过**（writing-great-skills 和 shared-language 新建时已修剪）。

- [ ] **Step 9.1: 对每个技能逐句应用 no-op 测试**

对每个技能文件：

1. Read 文件
2. 逐句问："删了这行，agent 的行为会变吗？"
3. 会变 → 保留；不会变 → 删除
4. 检测重复描述 → 保留第一次出现，删除后续

常见可删除模式：
- "Be thorough and careful" / "Make sure to..."（agent 不需要被鼓励）
- "This is crucial because..."（废话前缀）
- 重复的步骤说明（已在其他段描述过）
- 过度解释性的段落（代码示例已经说明了用途）

- [ ] **Step 9.2: 验证修剪后每个技能仍可加载**

对每个修改的技能，确认：
- Frontmatter 完整（name, description）
- description 仍触发正确的使用场景
- 核心工作流步骤完整

- [ ] **Step 9.3: 提交**

```powershell
git add skills/
git commit -m "refactor(skills): no-op trim across all 15 skills — remove sentences that don't change agent behavior"
```

---

### Task 10: 全面更新 README.md

**文件:**
- Modify: `README.md` — 全量同步

- [ ] **Step 10.1: 读取当前 README**

- [ ] **Step 10.2: 更新仓库树**

```markdown
├── .ai/
│   └── calibration.yml       # code-review severity calibration
├── agents/                   # 10 specialized agents
│   ├── orchestrator.md       # primary: intent gate + model-aware routing
│   ├── planner.md            # pro: architecture & planning
│   ├── deep-worker.md        # pro: heavy implementation
│   ├── oracle.md             # pro: deep code analysis (RO)
│   ├── reviewer.md           # pro: dual-axis code review (RO)
│   ├── consultant.md         # pro: brainstorming & advice
│   ├── ui-builder.md         # pro: frontend & UI
│   ├── explore.md            # flash: codebase search (RO)
│   ├── librarian.md          # flash: external research (RO)
│   └── light-orchestrator.md # flash: simple edits
├── skills/                   # 17 on-demand skills
│   ├── code-review/          # dual-axis parallel review + calibration
│   ├── codemap/              # structured repo maps
│   ├── gh-cli/               # GitHub CLI v2.96+ reference
│   ├── gh-skill/             # gh skill management
│   ├── git-master/           # advanced git operations
│   ├── git-release/          # tagged releases
│   ├── handoff/              # session compression
│   ├── opencode-config/      # meta: config authoring for this repo
│   ├── reflect/              # continuous improvement
│   ├── remove-deadcode/      # dead code detection & removal
│   ├── security-review/      # security audit checklist
│   ├── shared-language/      # domain glossary (saves tokens)
│   ├── simplify/             # behavior-preserving simplification
│   ├── spec-workflow/        # spec-driven development
│   ├── verification-planning/# pre-implementation verification path
│   ├── verify-with-docs/     # retrieval-first API verification
│   └── writing-great-skills/ # skill authoring discipline
├── opencode.jsonc            # main config (18 commands)
├── AGENTS.md                 # global rules (~210 lines)
├── dcp.jsonc                 # DCP context pruning config
├── LICENSE
└── README.md
```

- [ ] **Step 10.3: 更新 Agent 表**

反映 reviewer 改为双轴 code-review，其余 9 个不变。

- [ ] **Step 10.4: 更新 Skill 表**

删除 3 个（deepwork / conventional-commits / diagnose），新增 2 个（writing-great-skills / shared-language），code-review 描述改为 "双轴并行（Std+Spec）"。

- [ ] **Step 10.5: 更新命令表**

29 → 18 命令，删除 13 个，新增 spec-propose 和 spec-apply，/commit 更新为内联 CC 格式，/simplify 更新为 oracle 分析 + writer 应用。

- [ ] **Step 10.6: 更新统计数据**

- DCP 阈值：45K-85K → 35K-75K
- AGENTS.md 行数：更新为实际行数
- superpowers 技能数量：核实后更新

- [ ] **Step 10.7: 追加 v21 迭代记录**

在 README 迭代日志中追加：

```markdown
| 2026-07-31 | v21 | 全面瘦身重构：技能 18→17、命令 29→18(-38%)、AGENTS.md 压缩 -8%、技能总行数 -24%。code-review 双轴并行 + 校准文件机制。新增 writing-great-skills 和 shared-language。借鉴 pi/deepreview/mattpocock 等 6 个仓库。 |
```

- [ ] **Step 10.8: 端到端读取 README**

- [ ] **Step 10.9: 提交**

```powershell
git add README.md
git commit -m "docs(readme): sync for v21 — updated agent/skill/command tables, stats, iteration log"
```

---

### Task 11: 最终验证与总结

**文件:** 所有改动文件

- [ ] **Step 11.1: 全局一致性检查**

```powershell
# 1. 确认技能目录数量
(Get-ChildItem -Path skills -Directory).Count
# 预期: 17

# 2. 确认 agent 文件数量
(Get-ChildItem -Path agents -Filter *.md).Count
# 预期: 10

# 3. 确认 opencode.jsonc 命令数
# 手动检查：command 对象应有 18 个键

# 4. 确认 AGENTS.md 行数
(Get-Content AGENTS.md).Count
# 预期: ≤215

# 5. 确认无残留引用已删除技能
grep -r "conventional-commits" --include="*.md" --include="*.jsonc"
grep -r "deepwork" --include="*.md" --include="*.jsonc" --exclude="docs/*"
grep -r "diagnose" --include="*.md" --include="*.jsonc" --exclude="docs/*" --exclude="skills/diagnose/*"
# 预期: docs/ 下设计文档中有引用（可接受），其他位置无
```

- [ ] **Step 11.2: 验证所有新增文件可读且格式正确**

Read 每个新增文件确认无语法错误。

- [ ] **Step 11.3: 汇总统计**

对照 spec 的 Token 预算对比表：
- 技能数量: 17 ✓（目标 17）
- 命令数量: 18 ✓（目标 18）
- AGENTS.md 行数: 实际行数，确认 ≤215

- [ ] **Step 11.4: 最终提交**

```powershell
git add -A
git diff --cached --stat
git commit -m "chore: v21 final verification — consistency checks, residual reference cleanup"
```
