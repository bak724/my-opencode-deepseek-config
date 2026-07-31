# OpenCode v21 全面重构设计

> 日期: 2026-07-31
> 目标: 在严格控制 token 消耗的前提下，让 OpenCode 与 DeepSeek 模型深度结合达到最优开发效果

## 一、背景与动机

当前 v20 配置经过多次迭代已高度自洽，但存在以下改进空间：

1. **上下文膨胀**: AGENTS.md 与 orchestrator.md 存在大量路由规则重复注入；29 个命令常驻上下文
2. **技能质量**: 部分技能（conventional-commits、diagnose）功能重叠或可简化；code-review 为单轴审查，未利用并行优势
3. **数值漂移**: README、dcp.jsonc 注释与实际配置不同步（6 处 bug）
4. **元能力缺失**: 缺乏技能写作规范（导致后续技能膨胀）、缺乏共享领域语言机制

## 二、设计原则

- **减法为主，加法为辅**: 总 token 预算下降 ≥20%
- **一技能一使命**: 每个技能专注单一目标，不重叠
- **双轴并行审查**: Standards + Spec 独立运行，互不污染
- **按需加载**: 编排型元技能通过引用加载，不常驻描述

## 三、技能变更

### 删除（3 个）

| 技能 | 理由 | 替代 |
|------|------|------|
| `deepwork` | 与 spec-workflow 功能重叠（都是分阶段工作流） | spec-workflow |
| `conventional-commits` | 290 行独立技能，本质是格式规则 | 内嵌到 writing-great-skills 和 /commit 命令 |
| `diagnose` | 要求先加载 systematic-debugging，本质是包装器 | systematic-debugging 直接使用 |

### 新增（2 个）

| 技能 | 行数 | 触发条件 | 核心价值 |
|------|------|----------|----------|
| `writing-great-skills` | ~100 | 创建/编辑技能 | no-op 修剪纪律、正向表述、完成标准、conventional-commits 格式参考 |
| `shared-language` | ~80 | 术语/CONTEXT.md | 浓缩领域行话为共享语言，降低 agent 思考 token |

> 注：code-review 的双轴检查清单（Standards 轴 + Spec 轴）作为内联内容嵌入父技能，通过子代理分发时传入，不创建独立技能文件。避免技能计数膨胀的同时，保持双轴并行审查能力。

### 重构（3 个）

| 技能 | 当前行数 | 目标行数 | 改动 |
|------|---------|---------|------|
| `code-review` | 314 | ~280 | 双轴并行调度（内联 Std/Spec 检查清单）+ 有效大小分级 + 校准文件机制 + review→fix 循环 + PR 行级评论 |
| `spec-workflow` | ~350 | ~280 | 并入 OpenSpec Update vs 新建启发式；no-op 修剪 |
| `verify-with-docs` | ~130 | ~110 | 新增 references 本地优先规则；no-op 修剪 |

## 四、命令变更

### 删除（13 个）

`/review-loop` `/docs` `/learn` `/explore` `/skill` `/verify-plan` `/deepwork` `/propose` `/apply` `/update` `/archive` → 精简为 `/spec-propose` + `/spec-apply`

### 保留（16 个）

`/deep` `/quick` `/ui` `/review` `/review-pr` `/plan` `/search` `/oracle` `/consult` `/commit` `/release` `/reflect` `/handoff` `/codemap` `/simplify` `/rmslop`

### 新增（2 个）

`/spec-propose` `/spec-apply`

**总计: 29 → 18（-38%）**

## 五、AGENTS.md 精简

- **删除**: 意图门控路由表、Task Categories 表、Model-Aware Routing 表、Agent Directory 表、Fallback Chains 表——均已在 orchestrator.md 中维护
- **压缩 Code Style**: 15 条 → 5 条核心规则
- **新增**: 3 条 git 安全纪律（不 `git add -A`/`reset --hard`、仅 stage 本会话文件）
- **最终行数**: 227 → ~165（-27%）

## 六、Bug 修复

| # | 问题 | 位置 | 修复 |
|---|------|------|------|
| 1 | README DCP 阈值 45K-85K 应为 35K-75K | README.md | 修正 |
| 2 | dcp.jsonc 注释 85K 与值 75000 矛盾 | dcp.jsonc | 注释改为 75K |
| 3 | README AGENTS.md 行数声称过时 | README.md | 更新 |
| 4 | /simplify 路由到只读 oracle 却要求编辑 | opencode.jsonc | oracle 分析 → light-orchestrator 应用 |
| 5 | opencode-config 技能引用 opencode.json | opencode-config/SKILL.md | 改为 opencode.jsonc |
| 6 | README 仓库树缺 LICENSE | README.md | 补上 |

## 七、新增文件

| 文件 | 说明 |
|------|------|
| `.ai/calibration.yml` | code-review 严重度校准模板（gitignored，零常驻成本） |

## 八、Agent 变更

| Agent | 变更 |
|-------|------|
| `reviewer` | prompt 改为加载重构后的 code-review 父技能（双轴并行） |

其余 9 个 agent 无结构变更。

## 九、Token 预算对比

| 范围 | 当前 | 目标 | 变化 |
|------|------|------|------|
| 技能数量 | 18 | 17（-1） | — |
| 技能总行数 | ~3,500 | ~2,650 | **-24%** |
| 命令数量 | 29 | 18 | **-38%** |
| AGENTS.md | 227 行 | ~165 行 | **-27%** |
| 常驻上下文 | ~4.5KB/会话 | ~3.5KB/会话 | **-22%** |

## 十、实施阶段

| Phase | 内容 | 预估文件数 |
|-------|------|-----------|
| 1 删除 | 删 3 技能 + AGENTS.md 冗余块 + 13 命令 | ~5 |
| 2 Bug 修复 | 6 处同步修复 | ~4 |
| 3 新增技能 | 2 新技能（writing-great-skills、shared-language） + .ai/calibration.yml | 3 |
| 4 重构技能 | code-review + spec-workflow + verify-with-docs | 3 |
| 5 命令重构 | opencode.jsonc + reviewer agent | 2 |
| 6 AGENTS.md | 去重→压缩→加 git 纪律 | 1 |
| 7 no-op 修剪 | 全部技能逐句修剪 | ~15 |
| 8 README 更新 | 全面同步 + v21 迭代 | 1 |
