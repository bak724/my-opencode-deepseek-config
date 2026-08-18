# My OpenCode × DeepSeek Config

[简体中文](README.md) | **繁體中文** | [English](README.en-US.md) | [Русский](README.ru-RU.md) | [Français](README.fr-FR.md) | [Deutsch](README.de-DE.md) | [Español](README.es-ES.md) | [Português](README.pt-BR.md) | [日本語](README.ja-JP.md) | [한국어](README.ko-KR.md)

**OpenCode × DeepSeek 最優配置** —— 在 OpenCode 多 Agent 框架下，將 DeepSeek V4 雙模型（Pro + Flash）的能力發揮到極致的配置方案。核心理念：**Token 效率優先，用最小的上下文成本達到最好的開發效果**。

## 當前配置概覽

- 預設主 Agent：`orchestrator`
- 主模型：`deepseek/deepseek-v4-pro`，輕量模型：`deepseek/deepseek-v4-flash`
- 代理層級：`subagent_depth: 3`（支援 3 級代理巢狀）
- 模型隔離：`enabled_providers: ["deepseek"]` 單鎖
- 對話分享：關閉（`share: "disabled"`）；快照：開啟（`snapshot: true`）
- 權限基線：預設放行，破壞性 bash 命令設為 `ask`；`.env` 類敏感檔案 `deny`；外部目錄 `ask`；唯讀 Agent 的 bash 白名單（預設 deny 全部 + 僅放行唯讀子命令）
- 上下文壓縮：內建 compaction（opencode.jsonc）管自動觸發 + prune 裁舊工具輸出，DCP（dcp.jsonc）管主動去重 + 壓縮閾值，兩者互補
- 全域規則：`AGENTS.md`（核心原則、任務拒絕契約、自我驗證、反模式等；上下文/Token 紀律已下沉到 `orchestrator`）
- 技能：`skills/` 目錄下 **23 個** `SKILL.md` 技能，透過原生 `skill` 工具按需載入
- 外掛：`superpowers`（v6.3.0，過程型技能）、`@tarquinen/opencode-dcp`（智慧上下文裁剪）

## DeepSeek 模型配置

### 前置條件

- OpenCode ≥ v1.18.x（DeepSeek provider 為內建）
- DeepSeek API Key：[platform.deepseek.com/api_keys](https://platform.deepseek.com/api_keys) 申請

### 方式一：TUI 互動式配置（推薦）

```bash
opencode
# 在 TUI 中輸入: /connect → 選擇 DeepSeek → 貼上 API Key
# 然後: /models → 選擇 deepseek-v4-pro
```

API Key 會自動持久化到 `~/.local/share/opencode/auth.json`。

### 方式二：環境變數

Windows PowerShell:
```powershell
$env:DEEPSEEK_API_KEY="sk-your-key-here"
opencode
```

永久設定：將 `DEEPSEEK_API_KEY` 新增到系統環境變數。

### Provider 配置參考

```jsonc
{
  "model": "deepseek/deepseek-v4-pro",
  "small_model": "deepseek/deepseek-v4-flash",
  "enabled_providers": ["deepseek"]
}
```

如需為 Pro 模型啟用 thinking/reasoning，可在 `provider` 中追加：

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

> **模型 ID 命名規則**：`provider_id/model_id`，即 `deepseek/deepseek-v4-pro` 和 `deepseek/deepseek-v4-flash`。

## 安裝部署

### 方式一：複製 + 環境變數（推薦，跨平台通用）

```bash
git clone https://github.com/znlgis/my-opencode-deepseek-config.git
```

然後將 `OPENCODE_CONFIG_DIR` 指向儲存庫內的 `opencode/` 子目錄即可使用。

**Windows（PowerShell）** —— 永久生效：

```powershell
[Environment]::SetEnvironmentVariable("OPENCODE_CONFIG_DIR", "D:\path\to\my-opencode-deepseek-config\opencode", "User")
```

**Windows（PowerShell）** —— 暫時生效（僅目前工作階段）：

```powershell
$env:OPENCODE_CONFIG_DIR = "D:\path\to\my-opencode-deepseek-config\opencode"
opencode
```

**Linux / macOS** —— 追加到 `~/.bashrc` 或 `~/.zshrc`：

```bash
export OPENCODE_CONFIG_DIR="$HOME/path/to/my-opencode-deepseek-config/opencode"
```

### 方式二：符號連結到全域配置目錄

**Windows（PowerShell，需系統管理員）：**

```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.config"
New-Item -ItemType SymbolicLink -Path "$env:USERPROFILE\.config\opencode" -Target "D:\path\to\my-opencode-deepseek-config\opencode"
```

**Linux / macOS：**

```bash
ln -s /path/to/my-opencode-deepseek-config/opencode ~/.config/opencode
```

> **相容性說明**：`~/.config/opencode` 是 OpenCode 的標準全域配置路徑。本儲存庫的 `opencode/` 子目錄內含 `agents/`、`skills/`、`AGENTS.md` 等檔案，佈局完全遵循 OpenCode 約定，透過環境變數或符號連結指向後即可被自動識別。

### 驗證安裝

啟動 OpenCode 確認：
1. `/models` → 當前模型為 `deepseek/deepseek-v4-pro`
2. Agent 列表應能看到 `orchestrator`、`planner`、`deep-worker` 等 10 個 Agent
3. 輸入任意請求，Orchestrator 自動分析意圖並路由

## 模型分工

本儲存庫嚴格限制在 DeepSeek V4 雙模型內分工，不引入其他模型：

| 模型 | 用途 |
| --- | --- |
| `deepseek/deepseek-v4-pro` | 規劃、架構、根因分析、程式碼審查、重型實作、主控排程 |
| `deepseek/deepseek-v4-flash` | 快速探索、外部檢索、輕量任務、簡單編輯 |

### 路由策略

- **Flash 優先**：搜尋、尋找、簡單編輯等明確定義的任務優先走 flash agent
- **Pro 專注推理**：規劃、分析、審查、複雜實作——只用 pro
- **自動升級**：flash agent 無法勝任時自動升級到 pro（帶完整上下文）

## Agent 結構

### Primary Agent

| Agent | 模型 | 作用 |
| --- | --- | --- |
| `orchestrator` | v4-pro | 預設入口：意圖門控（Intent Gate）+ 模型感知路由 + 後備鏈 |

### Subagents

| Agent | 模型 | 權限 | 作用 |
| --- | --- | --- | --- |
| `planner` | v4-pro | 讀寫 | 規劃、架構、拆解任務 |
| `deep-worker` | v4-pro | 讀寫 | 重型實作、多檔案改動、複雜除錯 |
| `oracle` | v4-pro | **唯讀** | 根因分析、深度理解程式碼 |
| `reviewer` | v4-pro | **唯讀** | 雙軸程式碼審查（規範 + 規約）+ 嚴重度校準 |
| `ui-builder` | v4-pro | 讀寫 | 前端與 UI 相關任務 |
| `consultant` | v4-pro | 讀寫 | 方案討論、最佳實踐建議 |
| `explore` | v4-flash | **唯讀** | 程式碼庫搜尋、並行探索 |
| `librarian` | v4-flash | **唯讀** | 文件檢索、Web 搜尋 |
| `light-orchestrator` | v4-flash | 讀寫 | 輕量任務、單檔案編輯 |

> `deep-worker` 和 `light-orchestrator` 遵循「禁止研究、禁止委託」原則——執行而非探索，上下文由 orchestrator 提供。
>
> 唯讀 Agent（`oracle`/`reviewer`/`explore`/`librarian`）真唯讀化：`edit: deny` + bash 白名單（預設 deny 全部，僅放行 `git status/diff/log/show/blame/grep`、`rg` 等唯讀子命令；`oracle`/`reviewer` 另允許 `gh pr view/diff`、`gh issue view`、`gh api` 以支援 `/review-pr` 回帖）。

## 快捷命令

### Agent 路由命令

| 命令 | Agent | 用途 |
| --- | --- | --- |
| `/deep` | `deep-worker` | 重型實作、多檔案改動 |
| `/quick` | `light-orchestrator` | 輕量任務、單檔案編輯 |
| `/ui` | `ui-builder` | 前端/UI 工作 |
| `/review` | `reviewer`（code-review） | 雙軸並行審查（規範+規約）+ 嚴重度校準 |
| `/review-pr` | `reviewer`（code-review + gh-cli） | 審查 PR 並回帖到 GitHub |
| `/plan` | `planner` | 制定計畫、技術方案 |
| `/search` | `librarian` | 外部搜尋、查文件 |
| `/oracle` | `oracle` | 深度分析、問題溯源 |
| `/consult` | `consultant` | 諮詢、對比、建議 |

### 操作命令

| 命令 | Agent | 用途 |
| --- | --- | --- |
| `/commit` | `light-orchestrator` | 生成 Conventional Commits 提交訊息（內聯格式） |
| `/release` | `deep-worker`（git-release） | 準備 Tag 發布 |
| `/reflect` | `oracle`（reflect） | 發現摩擦 → 提出配置最佳化 |
| `/handoff` | `light-orchestrator`（handoff） | 壓縮對話為交接文件 |

### 內聯命令

| 命令 | Agent | 用途 |
| --- | --- | --- |
| `/codemap` | `explore`（codemap） | 生成儲存庫結構圖 |
| `/simplify` | `oracle`（simplify）→ `light-orchestrator` | oracle 分析 → light-orchestrator 應用簡化 |
| `/rmslop` | `deep-worker`（remove-deadcode） | 清理死程式碼和 AI slop |

### 規約命令

| 命令 | Agent | 用途 |
| --- | --- | --- |
| `/spec-propose` | `planner`（spec-workflow） | 探索程式碼 → 起草變更提案 |
| `/spec-apply` | `deep-worker`（spec-workflow） | 按 tasks.md 逐一實作 → 自動歸檔 |

## 技能（Skills）

OpenCode 透過原生 `skill` 工具按需暴露技能——Agent 只在需要時才載入，不佔用上下文。

| Skill | 作用 |
| --- | --- |
| `code-review` | 省 token 多維程式碼審查：按維度+嚴重度分級報告，一致點標最高置信度，deepreview 自我證偽，從不擅自改碼 |
| `codemap` | 生成帶標註的儲存庫結構圖，快速定向，節省探索 token |
| `gh-cli` | GitHub CLI v2.97+ 參考：分頁、儲存庫定位、discussions/projects/rulesets/skills、rate limit、gh-aw agentic CI、gh api 回退 |
| `git-master` | 進階 Git 操作：rebase、squash、fixup、bisect、reflog、程式碼考古、worktree |
| `git-release` | Tag 發布：發布說明、SemVer 推斷、gh release 命令 |
| `resolving-merge-conflicts` | 逐 hunk 解析合併衝突：追溯原始意圖、永不發明新行為、永不 --abort |
| `handoff` | 壓縮對話為交接文件（路徑引用，不複製內容） |
| `opencode-config` | 編寫和維護本儲存庫 OpenCode 配置（agents/skills/commands/permissions） |
| `reflect` | 持續改進：發現摩擦 → 提出最小可維護修復 |
| `remove-deadcode` | 安全尋找並刪除死程式碼，刪除前經工具鏈/LSP 驗證 |
| `security-review` | 合併前安全審查（注入/XSS/SSRF/金鑰/反序列化/路徑穿越），只報不改 |
| `shared-language` | 建構領域術語表（CONTEXT.md），大幅節省 token |
| `simplify` | 行為保持的程式碼簡化（oracle 分析 → 應用） |
| `spec-workflow` | 輕量規約驅動變更：proposal → specs → design → tasks → archive |
| `verification-planning` | 實作前規劃最窄驗證路徑 |
| `verify-with-docs` | 編碼前核對 API 文件，檢索優先，防幻覺 |
| `grilling` | 需求對齊訪談：一次一問、多選優先，歧義收斂後再動手 |
| `tech-debt-audit` | 9 維度技術債審計（死程式碼/重複/命名漂移/複雜度/依賴/錯誤處理/測試/文件/安全），唯讀報告不改碼 |
| `wait-what` | 使用者訊息難懂時先一句話重述確認，再動手 |
| `writing-for-agents` | 寫給 agent 看的文件（skill/AGENTS.md/指針文件）的寫作槓桿 |
| `to-questionnaire` | 離通道一次性問卷（非同步填寫），區別於 grilling 的即時訪談 |
| `research` | 開放課題深調研，產出帶引用的 Markdown，區別於 verify-with-docs 的單點核對 |
| `wizard` | 人工逐步嚮導（bash 指令碼，`bash -n` 驗證），引導人類完成自身才能做的步驟 |

## 設計決策與迭代記錄

核心思路借鑑了 [oh-my-openagent](https://github.com/code-yeongyu/oh-my-openagent)（意圖門控、唯讀隔離、反模式）、[oh-my-opencode-slim](https://github.com/alvinunreal/oh-my-opencode-slim)（排程器優先、後備鏈、拒絕契約、提示詞快取安全、impact×confidence÷cost）、[anomalyco/opencode](https://github.com/anomalyco/opencode)（配置 Schema、技能體系）、[cli/cli](https://github.com/cli/cli)（gh v2.97 命令集、rate limit、gh-aw）、[OpenSpec](https://github.com/Fission-AI/OpenSpec)（delta specs、OPSX 動作流 update/verify/四問）、[mattpocock/skills](https://github.com/mattpocock/skills)（衝突解析紀律、交接文件）、[pi](https://github.com/earendil-works/pi)（先答後改、精簡回應、獨立對話收集）和 [deepreview](https://github.com/mechanai/deepreview)（novelty 分類收斂、有效大小路由、Points of Agreement）的優點，純配置實作，零額外依賴。

> **借鑑而非照搬**：過重的流水線只汲取輕量化設計理念；冗餘功能由現有 agents/skills 覆蓋，不新增。遵循「精簡優先於新增」原則，每次迭代都以淨減 token 為目標。
>
> **本輪（v28）機制來源**：DeepSeek 快取+thinking 紀律、scope-first+委派優先、原子 TODO 下沉進 AGENTS.md；新增 5 技能（wait-what/writing-for-agents/to-questionnaire/research/wizard）至 23 個；gh-cli 增補 4 條 GHSA 安全條目；code-review 融入 deepreview 自我證偽；刪除 .ai/calibration.yml（校準規則內聯進 code-review）。
>
> **評估後未採用**：mattpocock/skills 的其餘流程類技能（code-review、tdd、implement 等與 superpowers／現有技能重疊）；superpowers 無配置旋鈕，保持外掛字串形式注入。

### 迭代里程碑

自 v1 以來歷經 28 次迭代，持續對標上游儲存庫最佳實踐：

- **v1-v7（奠基）**：雙模型繫結、Agent 角色體系、意圖門控路由、AGENTS.md 全域規則、Skills 目錄、權限基線
- **v8-v15（審查+規約+契約）**：code-review 雙軸校準、spec-workflow、gh-cli 對齊、拒絕契約、後臺核查
- **v16-v22（持續瘦身）**：命令 29→18（-38%）、AGENTS.md 290→211（-27%）、逐句 no-op 修剪、Schema 校驗去死鍵
- **v23-v25（對齊+安全）**：整合 6 個上游儲存庫、gh-cli v2.97 轉義注入安全章節、procedure-driven 提示精化、DCP 視窗調優
- **v26（本輪瘦身）**：prune:true 與 tool_output 800/20480 收緊、DCP 切換 60%/30% 百分比閾值、grilling 引入替代 writing-great-skills、opencode-config 131→64 精簡、code-review 分級+validator、gh-cli 補 gh status、AGENTS.md 增 User Override、orchestrator 委託成本紀律、7 個 agent 檔案淨減 22 行
- **v27（刪除/遷移/新增）**：刪 batch_tool 死配置、唯讀 agent 無效 `write: deny`、bash 3 條冗餘；Context Management 段遷入 orchestrator 專屬小節；唯讀 agent bash 白名單、read 補 `.env`；新增 tech-debt-audit 技能；15 條技能 description 瘦身 30-40%；gh-cli 補 rate limit/gh skill 宿主/gh-aw 等 5 點、code-review 增 Points of Agreement、spec-workflow 補 update 兩問、orchestrator 增獨立對話收集+提示詞快取安全、deep-worker 增 impact×confidence÷cost
- **v28（紀律重構）**：快取+thinking 紀律、scope-first+委派優先、原子 TODO 下沉 AGENTS.md；新增 5 技能至 23 個；gh-cli 補 4 條 GHSA；code-review 融入 deepreview 自我證偽；刪除 .ai/calibration.yml（規則內聯進 code-review）；README 十語種同步

## 儲存庫結構

```text
├── opencode/                     # OpenCode 配置目錄（可獨立部署）
│   ├── agents/                   # 10 個專職 Agent
│   │   ├── orchestrator.md       # 主入口：意圖門控 + 模型感知路由
│   │   ├── planner.md            # pro：架構與規劃
│   │   ├── deep-worker.md        # pro：重型實作
│   │   ├── oracle.md             # pro：深度程式碼分析（唯讀）
│   │   ├── reviewer.md           # pro：雙軸程式碼審查（唯讀）
│   │   ├── consultant.md         # pro：方案討論與建議
│   │   ├── ui-builder.md         # pro：前端與 UI
│   │   ├── explore.md            # flash：程式碼庫搜尋（唯讀）
│   │   ├── librarian.md          # flash：外部檢索（唯讀）
│   │   └── light-orchestrator.md # flash：簡單編輯
│   ├── skills/                   # 23 個按需載入技能
│   │   ├── code-review/          # 雙軸並行審查 + 嚴重度校準
│   │   ├── codemap/              # 生成儲存庫結構圖
│   │   ├── gh-cli/               # GitHub CLI v2.97+ 參考 + 安全警告
│   │   ├── git-master/           # 進階 Git 操作
│   │   ├── git-release/          # Tag 發布
│   │   ├── handoff/              # 對話壓縮為交接文件
│   │   ├── opencode-config/      # 元技能：本儲存庫配置編寫
│   │   ├── reflect/              # 持續改進
│   │   ├── remove-deadcode/      # 死程式碼檢測與刪除
│   │   ├── resolving-merge-conflicts/ # 逐 hunk 衝突解析紀律
│   │   ├── security-review/      # 安全審查清單
│   │   ├── shared-language/      # 領域術語表（節省 token）
│   │   ├── simplify/             # 行為保持的程式碼簡化
│   │   ├── spec-workflow/        # 規約驅動開發
│   │   ├── tech-debt-audit/      # 技術債審計（9 維度，唯讀報告）
│   │   ├── verification-planning/ # 實作前驗證路徑規劃
│   │   ├── verify-with-docs/     # 檢索優先 API 驗證
│   │   ├── grilling/             # 需求對齊訪談
│   │   ├── research/             # 開放課題深調研（帶引用）
│   │   ├── to-questionnaire/     # 離通道一次性問卷
│   │   ├── wait-what/            # 難懂訊息先一句話重述確認
│   │   ├── wizard/               # 人工逐步嚮導（bash -n 驗證）
│   │   └── writing-for-agents/   # 面向 agent 的文件寫作
│   ├── opencode.jsonc            # 主配置（18 條命令）
│   ├── AGENTS.md                 # 全域規則
│   └── dcp.jsonc                 # DCP 上下文壓縮（DeepSeek 128K，60%/30% 百分比閾值）
├── README.md
├── LICENSE
└── README.*.md                   # 其他語言 README
```

## 使用指南

### 模式一：Orchestrator 自動路由（預設）

用自然語言描述需求，Orchestrator 自動分析意圖、選擇最合適的 Agent 和模型執行。

```text
「幫我排查這個登入介面的報錯」     → oracle 分析根因 → 返回診斷報告
「最佳化這段迴圈，效能太差了」     → oracle 分析 → deep-worker 實施最佳化
「這個 PR 幫我審查一下」           → reviewer 多維度審查 → 返回分級報告
「我想給使用者模組加個匯出功能」   → planner 制定方案 → deep-worker 實作
「React 19 的 use() API 怎麼用」  → librarian 查文件 → 返回簽名和範例
```

### 模式二：命令別名直達

| 場景 | 命令 |
| --- | --- |
| 複雜實作 / 多檔案改動 | `/deep` |
| 輕量修改 / 單檔案編輯 | `/quick` |
| 制定技術方案 / 架構設計 | `/plan` |
| 排查 Bug / 深度分析 | `/oracle` |
| 程式碼審查 | `/review` |
| 外部搜尋 / 查 API | `/search` |
| 前端 / UI 工作 | `/ui` |
| 方案討論 / 對比取捨 | `/consult` |
| 結構化除錯 | `/oracle` |

### 典型工作流程

**開發新功能（規約驅動）：**
```text
/spec-propose  → /spec-apply  → /review
```

**排查 Bug：**
```text
/oracle  → /deep  → /rmslop  → /commit
```

**程式碼審查：**
```text
/review-pr   ← 審查 PR + 自動回帖
/review      ← 雙軸並行審查
```

## 設計哲學

- **純配置驅動，零額外依賴** —— 所有能力由 `opencode.jsonc` + `agents/*.md` + `skills/*/SKILL.md` + `AGENTS.md` 實作
- **DeepSeek V4 雙模型極致利用** —— Pro 做推理與決策，Flash 做查詢與輕量執行
- **Token 效率優先** —— 路徑引用替代貼上檔案、技能按需載入、壓縮分級管理
- **外掛增效但不喧賓奪主** —— superpowers 提供過程紀律，DCP（dcp.jsonc）主動去重+壓縮閾值，內建 compaction（opencode.jsonc）自動觸發+prune 兜底
- **執行與探索分離** —— deep-worker/light-orchestrator 禁止研究/委託，explore/librarian 禁止修改
- **快取與 thinking 紀律** —— 靜態前綴穩定以命中 DeepSeek 提示詞快取；編碼任務 0 溫度；thinking 僅對推理任務開啟，簡單/檢索任務關閉
- **Scope First + Delegate Always** —— 先定範圍（2+ 步/多檔案/架構變更先走 planner），再委派執行，頂層 token 只留給路由與難題
- **原子 TODO** —— 多步任務先寫有序 TODO，逐條 in_progress→completed；格式 `path: action for scenario — verify by check`
- **持續改進** —— reflect 機制化發現摩擦、code-review 雙軸校準保證品質
