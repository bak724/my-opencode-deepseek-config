# My OpenCode × DeepSeek Config

**OpenCode × DeepSeek 最优配置** —— 在 OpenCode 多 Agent 框架下，将 DeepSeek V4 双模型（Pro + Flash）的能力发挥到极致的配置方案。核心理念：**Token 效率优先，用最小的上下文成本达到最好的开发效果**。

## 当前配置概览

- 默认主 Agent：`orchestrator`
- 主模型：`deepseek/deepseek-v4-pro`，轻量模型：`deepseek/deepseek-v4-flash`
- 代理层级：`subagent_depth: 3`（支持 3 级代理嵌套）
- 模型隔离：`enabled_providers: ["deepseek"]` + `disabled_providers` 双重锁
- 会话分享：关闭（`share: "disabled"`）；快照：开启（`snapshot: true`）
- 权限基线：默认放行，破坏性 bash 命令设为 `ask`；`.env` 类敏感文件 `deny`；外部目录 `ask`
- 上下文压缩：DCP 主动压缩（35K-75K 阈值）+ OpenCode 原生 compaction 兜底
- 全局规则：`AGENTS.md`（核心原则、任务拒绝契约、Token 效率、证据纪律、反模式等）
- 技能：`skills/` 目录下 **17 个** `SKILL.md` 技能，通过原生 `skill` 工具按需加载
- 插件：`superpowers`（14 个过程型技能）、`@tarquinen/opencode-dcp`（智能上下文裁剪）
- 实验功能：`batch_tool` 已默认开启

## DeepSeek 模型配置

### 前置条件

- OpenCode ≥ v1.14.24（DeepSeek provider 为内置）
- DeepSeek API Key：[platform.deepseek.com/api_keys](https://platform.deepseek.com/api_keys) 申请

### 方式一：TUI 交互式配置（推荐）

```bash
opencode
# 在 TUI 中输入: /connect → 选择 DeepSeek → 粘贴 API Key
# 然后: /models → 选择 deepseek-v4-pro
```

API Key 会自动持久化到 `~/.local/share/opencode/auth.json`。

### 方式二：环境变量

Windows PowerShell:
```powershell
$env:DEEPSEEK_API_KEY="sk-your-key-here"
opencode
```

永久设置：将 `DEEPSEEK_API_KEY` 添加到系统环境变量。

### Provider 配置参考

```jsonc
{
  "model": "deepseek/deepseek-v4-pro",
  "small_model": "deepseek/deepseek-v4-flash",
  "enabled_providers": ["deepseek"],
  "disabled_providers": ["openai", "anthropic", "google", "openrouter"]
}
```

如需为 Pro 模型启用 thinking/reasoning，可在 `provider` 中追加：

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

> **模型 ID 命名规则**：`provider_id/model_id`，即 `deepseek/deepseek-v4-pro` 和 `deepseek/deepseek-v4-flash`。

## 安装部署

### 方式一：克隆到全局配置目录（推荐）

**Windows（PowerShell）：**

```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.config"
git clone https://github.com/<your-username>/my-opencode-deepseek-config.git "$env:USERPROFILE\.config\opencode"
```

**Linux / macOS：**

```bash
git clone https://github.com/<your-username>/my-opencode-deepseek-config.git ~/.config/opencode
```

> **兼容性说明**：`~/.config/opencode` 是 OpenCode 的标准全局配置路径。本仓库的 `agents/`、`skills/`、`AGENTS.md` 等文件遵循 OpenCode 的约定布局，克隆后无需额外配置即可被自动识别。

### 方式二：克隆到任意位置 + 环境变量

```powershell
$env:OPENCODE_CONFIG_DIR = "D:\path\to\opencode-config"
opencode
```

### 验证安装

启动 OpenCode 确认：
1. `/models` → 当前模型为 `deepseek/deepseek-v4-pro`
2. Agent 列表应能看到 `orchestrator`、`planner`、`deep-worker` 等 10 个 Agent
3. 输入任意请求，Orchestrator 自动分析意图并路由

## 模型分工

本仓库严格限制在 DeepSeek V4 双模型内分工，不引入其他模型：

| 模型 | 用途 |
| --- | --- |
| `deepseek/deepseek-v4-pro` | 规划、架构、根因分析、代码审查、重型实现、主控调度 |
| `deepseek/deepseek-v4-flash` | 快速探索、外部检索、轻量任务、简单编辑 |

### 路由策略

- **Flash 优先**：搜索、查找、简单编辑等明确定义的任务优先走 flash agent
- **Pro 专注推理**：规划、分析、审查、复杂实现——只用 pro
- **自动升级**：flash agent 无法胜任时自动升级到 pro（带完整上下文）

## Agent 结构

### Primary Agent

| Agent | 模型 | 作用 |
| --- | --- | --- |
| `orchestrator` | v4-pro | 默认入口，意图门控、任务分类（6 类）、模型感知路由 |

### Subagents

| Agent | 模型 | 权限 | 作用 |
| --- | --- | --- | --- |
| `planner` | v4-pro | 读写 | 规划、架构、拆解任务 |
| `deep-worker` | v4-pro | 读写 | 重型实现、多文件改动、复杂调试 |
| `oracle` | v4-pro | **只读** | 根因分析、深度理解代码 |
| `reviewer` | v4-pro | **只读** | 双轴代码审查（规范 + 规约）+ 严重度校准 |
| `ui-builder` | v4-pro | 读写 | 前端与 UI 相关任务 |
| `consultant` | v4-pro | 读写 | 方案讨论、最佳实践建议 |
| `explore` | v4-flash | **只读** | 代码库搜索、并行探索 |
| `librarian` | v4-flash | **只读** | 文档检索、Web 搜索 |
| `light-orchestrator` | v4-flash | 读写 | 轻量任务、单文件编辑 |

> `deep-worker` 和 `light-orchestrator` 遵循"禁止研究、禁止委托"原则——执行而非探索，上下文由 orchestrator 提供。

## 快捷命令

### Agent 路由命令

| 命令 | Agent | 用途 |
| --- | --- | --- |
| `/deep` | `deep-worker` | 重型实现、多文件改动 |
| `/quick` | `light-orchestrator` | 轻量任务、单文件编辑 |
| `/ui` | `ui-builder` | 前端/UI 工作 |
| `/review` | `reviewer`（code-review） | 双轴并行审查（规范+规约）+ 严重度校准 |
| `/review-pr` | `reviewer`（code-review + gh-cli） | 审查 PR 并回帖到 GitHub |
| `/plan` | `planner` | 制定计划、技术方案 |
| `/search` | `librarian` | 外部搜索、查文档 |
| `/oracle` | `oracle` | 深度分析、问题溯源 |
| `/consult` | `consultant` | 咨询、对比、建议 |

### 操作命令

| 命令 | Agent | 用途 |
| --- | --- | --- |
| `/commit` | `light-orchestrator` | 生成 Conventional Commits 提交信息（内联格式） |
| `/release` | `deep-worker`（git-release） | 准备 Tag 发布 |
| `/reflect` | `oracle`（reflect） | 发现摩擦 → 提出配置优化 |
| `/handoff` | `light-orchestrator`（handoff） | 压缩会话为交接文档 |

### 内联命令

| 命令 | Agent | 用途 |
| --- | --- | --- |
| `/codemap` | `explore`（codemap） | 生成仓库结构图 |
| `/simplify` | `oracle`（simplify）→ `light-orchestrator` | oracle 分析 → light-orchestrator 应用简化 |
| `/rmslop` | `deep-worker`（remove-deadcode） | 清理死代码和 AI slop |

### 规约命令

| 命令 | Agent | 用途 |
| --- | --- | --- |
| `/spec-propose` | `planner`（spec-workflow） | 探索代码 → 起草变更提案 |
| `/spec-apply` | `deep-worker`（spec-workflow） | 按 tasks.md 逐一实现 → 自动归档 |

## 技能（Skills）

OpenCode 通过原生 `skill` 工具按需暴露技能——Agent 只在需要时才加载，不占用上下文。

| Skill | 作用 |
| --- | --- |
| `code-review` | 双轴并行审查（规范 + 规约）+ 严重度校准 |
| `codemap` | 生成带标注的仓库结构图，节省探索 token |
| `gh-cli` | GitHub CLI v2.96+ 全面参考（Issues 2.0、copilot、agent-task） |
| `gh-skill` | 发现、安装、更新、发布 Agent 技能 |
| `git-master` | 高级 Git 操作：rebase、squash、bisect、reflog、worktree |
| `git-release` | Tag 发布：SemVer 推断、发布说明、gh release 命令 |
| `handoff` | 压缩会话为交接文档（路径引用，不复制内容） |
| `opencode-config` | 编写和维护 OpenCode 配置 |
| `reflect` | 持续改进：发现摩擦 → 提出最小修复 |
| `remove-deadcode` | 安全查找并删除死代码，删除前 LSP 验证 |
| `security-review` | 合并前对 diff 做安全审查 |
| `shared-language` | 构建领域术语表，大幅节省上下文 token |
| `simplify` | 行为保持的代码简化（oracle 分析 → light-orchestrator 应用） |
| `spec-workflow` | 轻量规约驱动变更（propose → design → tasks → implement → archive） |
| `verification-planning` | 实现前规划最窄验证路径 |
| `verify-with-docs` | 编码前核对 API 文档，检索优先，防止幻觉 |
| `writing-great-skills` | 技能编写规范：无操作裁剪、正向表述、完成标准 |

## 设计决策与迭代记录

核心思路借鉴了 [oh-my-openagent](https://github.com/code-yeongyu/oh-my-openagent)（意图门控、只读隔离、反模式）、[oh-my-opencode-slim](https://github.com/alvinunreal/oh-my-opencode-slim)（调度器优先、后备链、拒绝契约）、[anomalyco/opencode](https://github.com/anomalyco/opencode)（配置 Schema、技能体系）、[cli/cli](https://github.com/cli/cli)（gh 完整命令集）、[OpenSpec](https://github.com/Fission-AI/OpenSpec)（delta specs、变更提案更新）、[mattpocock/skills](https://github.com/mattpocock/skills)（交接文档、结构化调试）和 [deepreview](https://github.com/mechanai/deepreview)（熵扫描、收敛检查）的优点，纯配置实现，零额外依赖。

> **借鉴而非照搬**：过重的流水线只汲取轻量化设计理念；冗余功能由现有 agents/skills 覆盖，不新增。遵循"精简优先于新增"原则，每次迭代都以净减 token 为目标。

### 迭代里程碑

| 阶段 | 关键变更 |
| --- | --- |
| **v1-v7（奠基）** | 双模型绑定、agent 角色体系、意图门控/分类路由、AGENTS.md 全局规则、skills 目录与命令别名、权限基线 |
| **v8-v12（审查+规约）** | 增强 code-review（分级/自检/拒绝准则）、建立 spec-workflow（explore→propose→apply→archive）、新增 deepwork/reflect/verification-planning、gh-cli 对齐 v2.96+ |
| **v13-v15（契约+精简）** | AGENTS.md 新增 Evidence Discipline / Task Rejection Contract / stop condition；全量去重 agent prompt 与全局规则；补齐后台子 agent 错误核查 |
| **v16-v18（高效执行）** | 移除神话名称、合并路由表、gh-cli 扩至 Issues 2.0、spec-workflow 增加 verify + 决策框架 |
| **v19（对齐上游）** | 复核 6 个上游仓库；修正 `/review-pr` 逐行评论 Bug；code-review 路由从裸行数改为有效逻辑体量 |
| **v20（重构优化）** | `agent/`→`agents/` 对齐 OpenCode 推荐；AGENTS.md 精简 22%（292→229 行）；新增 `diagnose`（6 阶段调试）+ `handoff`（会话交接）技能；spec-workflow 增加 `/update`；code-review 增加熵扫描+收敛检查；agent prompt 去重 20% |
| **v21（全面瘦身重构）** | 技能 18→17（移除 deepwork/conventional-commits/diagnose，新增 writing-great-skills/shared-language）；命令 29→18（-38%）；AGENTS.md 227→212 行（-7%）；技能逐句 no-op 修剪。code-review 双轴并行 + 校准文件机制。借鉴 pi/deepreview/mattpocock 等 6 个仓库实战经验。 |

## 仓库结构

```text
├── .ai/
│   └── calibration.yml           # code-review 严重度校准
├── agents/                       # 10 个专职 Agent
│   ├── orchestrator.md           # 主入口：意图门控 + 模型感知路由
│   ├── planner.md                # pro：架构与规划
│   ├── deep-worker.md            # pro：重型实现
│   ├── oracle.md                 # pro：深度代码分析（只读）
│   ├── reviewer.md               # pro：双轴代码审查（只读）
│   ├── consultant.md             # pro：方案讨论与建议
│   ├── ui-builder.md             # pro：前端与 UI
│   ├── explore.md                # flash：代码库搜索（只读）
│   ├── librarian.md              # flash：外部检索（只读）
│   └── light-orchestrator.md     # flash：简单编辑
├── skills/                       # 17 个按需加载技能
│   ├── code-review/              # 双轴并行审查 + 严重度校准
│   ├── codemap/                  # 生成仓库结构图
│   ├── gh-cli/                   # GitHub CLI v2.96+ 参考
│   ├── gh-skill/                 # gh skill 管理
│   ├── git-master/               # 高级 Git 操作
│   ├── git-release/              # Tag 发布
│   ├── handoff/                  # 会话压缩为交接文档
│   ├── opencode-config/          # 元技能：本仓库配置编写
│   ├── reflect/                  # 持续改进
│   ├── remove-deadcode/          # 死代码检测与删除
│   ├── security-review/          # 安全审查清单
│   ├── shared-language/          # 领域术语表（节省 token）
│   ├── simplify/                 # 行为保持的代码简化
│   ├── spec-workflow/            # 规约驱动开发
│   ├── verification-planning/    # 实现前验证路径规划
│   ├── verify-with-docs/         # 检索优先 API 验证
│   └── writing-great-skills/     # 技能编写规范
├── opencode.jsonc                # 主配置（18 条命令）
├── AGENTS.md                     # 全局规则（~212 行）
├── dcp.jsonc                     # DCP 上下文压缩（DeepSeek 128K）
├── LICENSE
└── README.md
```

## 使用指南

### 模式一：Orchestrator 自动路由（默认）

用自然语言描述需求，Orchestrator 自动分析意图、选择最合适的 Agent 和模型执行。

```text
「帮我排查这个登录接口的报错」     → oracle 分析根因 → 返回诊断报告
「优化这段循环，性能太差了」         → oracle 分析 → deep-worker 实施优化
「这个 PR 帮我审查一下」             → reviewer 多维度审查 → 返回分级报告
「我想给用户模块加个导出功能」       → planner 制定方案 → deep-worker 实现
「React 19 的 use() API 怎么用」    → librarian 查文档 → 返回签名和示例
```

### 模式二：命令别名直达

| 场景 | 命令 |
| --- | --- |
| 复杂实现 / 多文件改动 | `/deep` |
| 轻量修改 / 单文件编辑 | `/quick` |
| 制定技术方案 / 架构设计 | `/plan` |
| 排查 Bug / 深度分析 | `/oracle` |
| 代码审查 | `/review` |
| 外部搜索 / 查 API | `/search` |
| 前端 / UI 工作 | `/ui` |
| 方案讨论 / 对比取舍 | `/consult` |
| 结构化调试 | `/oracle` |

### 典型工作流

**开发新功能（规约驱动）：**
```text
/spec-propose  → /spec-apply  → /review
```

**排查 Bug：**
```text
/oracle  → /deep  → /rmslop  → /commit
```

**代码审查：**
```text
/review-pr   ← 审查 PR + 自动回帖
/review      ← 双轴并行审查
```

## 设计哲学

- **纯配置驱动，零额外依赖** —— 所有能力由 `opencode.jsonc` + `agents/*.md` + `skills/*/SKILL.md` + `AGENTS.md` 实现
- **DeepSeek V4 双模型极致利用** —— Pro 做推理与决策，Flash 做查询与轻量执行
- **Token 效率优先** —— 路径引用替代粘贴文件、技能按需加载、压缩分级管理
- **插件增效但不喧宾夺主** —— superpowers 提供过程纪律，DCP 智能压缩替代简单截断
- **执行与探索分离** —— deep-worker/light-orchestrator 禁止研究/委托，explore/librarian 禁止修改
- **持续改进** —— reflect 机制化发现摩擦、code-review 双轴校准保证质量
