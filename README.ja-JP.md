# My OpenCode × DeepSeek Config

[简体中文](README.md) | [繁體中文](README.zh-TW.md) | [English](README.en-US.md) | [Русский](README.ru-RU.md) | [Français](README.fr-FR.md) | [Deutsch](README.de-DE.md) | [Español](README.es-ES.md) | [Português](README.pt-BR.md) | **日本語** | [한국어](README.ko-KR.md)

**OpenCode × DeepSeek 最適構成** —— OpenCode のマルチエージェントフレームワーク上で、DeepSeek V4 デュアルモデル（Pro + Flash）の能力を最大限に引き出す構成。中核理念：**トークン効率を最優先し、最小限のコンテキストコストで最高の開発成果を得る**。

## 現在の構成概要

- デフォルト主エージェント：`orchestrator`
- メインモデル：`deepseek/deepseek-v4-pro`、軽量モデル：`deepseek/deepseek-v4-flash`
- エージェント階層：`subagent_depth: 3`（3 段階のエージェントネストをサポート）
- モデル分離：`enabled_providers: ["deepseek"]` 単一ロック
- セッション共有：無効（`share: "disabled"`）；スナップショット：有効（`snapshot: true`）
- 権限ベースライン：デフォルト許可、破壊的な bash コマンドは `ask`；`.env` 系の機密ファイルは `deny`；外部ディレクトリは `ask`；読み取り専用エージェントは bash ホワイトリスト（デフォルト全 deny + 読み取り専用サブコマンドのみ許可）
- コンテキスト圧縮：内蔵 compaction（opencode.jsonc）が自動トリガー + prune で旧ツール出力を刈り込み、DCP（dcp.jsonc）が能動的な重複排除 + 圧縮閾値を担当。両者は相互補完
- グローバルルール：`AGENTS.md`（コア原則、タスク拒否契約、自己検証、アンチパターンなど；コンテキスト/トークン規律は `orchestrator` に集約）
- スキル：`skills/` ディレクトリに **23 個**の `SKILL.md` スキル。ネイティブの `skill` ツールでオンデマンド読み込み
- プラグイン：`superpowers`（v6.3.0、プロセス型スキル）、`@tarquinen/opencode-dcp`（インテリジェントなコンテキスト刈り込み）

## DeepSeek モデル構成

### 前提条件

- OpenCode ≥ v1.18.x（DeepSeek プロバイダーは内蔵）
- DeepSeek API キー：[platform.deepseek.com/api_keys](https://platform.deepseek.com/api_keys) で申請

### 方法 1：TUI 対話型設定（推奨）

```bash
opencode
# TUI で入力: /connect → DeepSeek を選択 → API キーを貼り付け
# その後: /models → deepseek-v4-pro を選択
```

API キーは `~/.local/share/opencode/auth.json` に自動的に永続化されます。

### 方法 2：環境変数

Windows PowerShell:
```powershell
$env:DEEPSEEK_API_KEY="sk-your-key-here"
opencode
```

永続設定：システム環境変数に `DEEPSEEK_API_KEY` を追加します。

### プロバイダー構成リファレンス

```jsonc
{
  "model": "deepseek/deepseek-v4-pro",
  "small_model": "deepseek/deepseek-v4-flash",
  "enabled_providers": ["deepseek"]
}
```

Pro モデルで thinking/reasoning を有効にする場合、`provider` に追記します：

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

> **モデル ID 命名規則**：`provider_id/model_id`、すなわち `deepseek/deepseek-v4-pro` と `deepseek/deepseek-v4-flash`。

## インストールとデプロイ

### 方法 1：クローン + 環境変数（推奨、クロスプラットフォーム共通）

```bash
git clone https://github.com/znlgis/my-opencode-deepseek-config.git
```

その後、`OPENCODE_CONFIG_DIR` をリポジトリ内の `opencode/` サブディレクトリに向ければ使用できます。

**Windows（PowerShell）** —— 永続的に有効：

```powershell
[Environment]::SetEnvironmentVariable("OPENCODE_CONFIG_DIR", "D:\path\to\my-opencode-deepseek-config\opencode", "User")
```

**Windows（PowerShell）** —— 一時的に有効（現在のセッションのみ）：

```powershell
$env:OPENCODE_CONFIG_DIR = "D:\path\to\my-opencode-deepseek-config\opencode"
opencode
```

**Linux / macOS** —— `~/.bashrc` または `~/.zshrc` に追記：

```bash
export OPENCODE_CONFIG_DIR="$HOME/path/to/my-opencode-deepseek-config/opencode"
```

### 方法 2：グローバル設定ディレクトリへのシンボリックリンク

**Windows（PowerShell、管理者権限が必要）：**

```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.config"
New-Item -ItemType SymbolicLink -Path "$env:USERPROFILE\.config\opencode" -Target "D:\path\to\my-opencode-deepseek-config\opencode"
```

**Linux / macOS：**

```bash
ln -s /path/to/my-opencode-deepseek-config/opencode ~/.config/opencode
```

> **互換性メモ**：`~/.config/opencode` は OpenCode の標準グローバル設定パス。本リポジトリの `opencode/` サブディレクトリには `agents/`、`skills/`、`AGENTS.md` などのファイルが含まれ、レイアウトは完全に OpenCode の規約に準拠しており、環境変数またはシンボリックリンクで指し示すだけで自動認識されます。

### インストール確認

OpenCode を起動して確認：
1. `/models` → 現在のモデルが `deepseek/deepseek-v4-pro` であること
2. Agent リストに `orchestrator`、`planner`、`deep-worker` など 10 個のエージェントが表示されること
3. 任意のリクエストを入力すると、Orchestrator が自動的に意図を分析してルーティングすること

## モデル分業

本リポジトリは DeepSeek V4 デュアルモデル内の分業に厳格に限定し、他のモデルは導入しません：

| モデル | 用途 |
| --- | --- |
| `deepseek/deepseek-v4-pro` | 計画、アーキテクチャ、根本原因分析、コードレビュー、重量級実装、統括スケジューリング |
| `deepseek/deepseek-v4-flash` | 高速探索、外部検索、軽量タスク、単純な編集 |

### ルーティング戦略

- **Flash 優先**：検索、ルックアップ、単純な編集など明確に定義されたタスクは flash エージェントを優先
- **Pro は推論に専念**：計画、分析、レビュー、複雑な実装——pro のみを使用
- **自動昇格**：flash エージェントで対応できない場合は自動的に pro へ昇格（完全なコンテキスト付き）

## Agent 構造

### プライマリエージェント

| Agent | モデル | 役割 |
| --- | --- | --- |
| `orchestrator` | v4-pro | デフォルト入口：意図ゲート（Intent Gate）+ モデル認識ルーティング + フォールバックチェーン |

### サブエージェント

| Agent | モデル | 権限 | 役割 |
| --- | --- | --- | --- |
| `planner` | v4-pro | 読み書き | 計画、アーキテクチャ、タスク分解 |
| `deep-worker` | v4-pro | 読み書き | 重量級実装、複数ファイル変更、複雑なデバッグ |
| `oracle` | v4-pro | **読み取り専用** | 根本原因分析、コードの深い理解 |
| `reviewer` | v4-pro | **読み取り専用** | 二軸コードレビュー（規範 + 規約）+ 深刻度キャリブレーション |
| `ui-builder` | v4-pro | 読み書き | フロントエンドと UI 関連タスク |
| `consultant` | v4-pro | 読み書き | 設計案の議論、ベストプラクティス提案 |
| `explore` | v4-flash | **読み取り専用** | コードベース検索、並列探索 |
| `librarian` | v4-flash | **読み取り専用** | ドキュメント検索、Web 検索 |
| `light-orchestrator` | v4-flash | 読み書き | 軽量タスク、単一ファイル編集 |

> `deep-worker` と `light-orchestrator` は「研究禁止・委任禁止」の原則に従う——実行であって探索ではない。コンテキストは orchestrator が提供する。
>
> 読み取り専用エージェント（`oracle`/`reviewer`/`explore`/`librarian`）は真の読み取り専用化：`edit: deny` + bash ホワイトリスト（デフォルト全 deny、`git status/diff/log/show/blame/grep`、`rg` などの読み取り専用サブコマンドのみ許可；`oracle`/`reviewer` はさらに `gh pr view/diff`、`gh issue view`、`gh api` を許可し `/review-pr` の返信をサポート）。

## ショートカットコマンド

### Agent ルーティングコマンド

| コマンド | Agent | 用途 |
| --- | --- | --- |
| `/deep` | `deep-worker` | 重量級実装、複数ファイル変更 |
| `/quick` | `light-orchestrator` | 軽量タスク、単一ファイル編集 |
| `/ui` | `ui-builder` | フロントエンド / UI 作業 |
| `/review` | `reviewer`（code-review） | 二軸並列レビュー（規範+規約）+ 深刻度キャリブレーション |
| `/review-pr` | `reviewer`（code-review + gh-cli） | PR をレビューして GitHub に返信 |
| `/plan` | `planner` | 計画策定、技術設計 |
| `/search` | `librarian` | 外部検索、ドキュメント調査 |
| `/oracle` | `oracle` | 深い分析、問題の追跡 |
| `/consult` | `consultant` | 相談、比較、提案 |

### 操作コマンド

| コマンド | Agent | 用途 |
| --- | --- | --- |
| `/commit` | `light-orchestrator` | Conventional Commits コミットメッセージ生成（インライン形式） |
| `/release` | `deep-worker`（git-release） | Tag リリースの準備 |
| `/reflect` | `oracle`（reflect） | 摩擦の発見 → 構成最適化の提案 |
| `/handoff` | `light-orchestrator`（handoff） | セッションを引き継ぎドキュメントに圧縮 |

### インラインコマンド

| コマンド | Agent | 用途 |
| --- | --- | --- |
| `/codemap` | `explore`（codemap） | リポジトリ構造図の生成 |
| `/simplify` | `oracle`（simplify）→ `light-orchestrator` | oracle が分析 → light-orchestrator が簡素化を適用 |
| `/rmslop` | `deep-worker`（remove-deadcode） | 死にコードと AI slop の削除 |

### 規約コマンド

| コマンド | Agent | 用途 |
| --- | --- | --- |
| `/spec-propose` | `planner`（spec-workflow） | コード探索 → 変更提案の起草 |
| `/spec-apply` | `deep-worker`（spec-workflow） | tasks.md に沿って逐次実装 → 自動アーカイブ |

## スキル（Skills）

OpenCode はネイティブの `skill` ツールでスキルをオンデマンド公開する——エージェントは必要な時だけ読み込み、コンテキストを占有しません。

| Skill | 役割 |
| --- | --- |
| `code-review` | トークン節約型の多次元コードレビュー：次元+深刻度で等級付け報告、一致点に最高確信度を付与、deepreview 自己反証、勝手にコードを書き換えない |
| `codemap` | 注釈付きリポジトリ構造図を生成、迅速な方向づけ、探索トークンの節約 |
| `gh-cli` | GitHub CLI v2.97+ リファレンス：ページネーション、リポジトリ特定、discussions/projects/rulesets/skills、rate limit、gh-aw agentic CI、gh api フォールバック |
| `git-master` | 高度な Git 操作：rebase、squash、fixup、bisect、reflog、コード考古学、worktree |
| `git-release` | Tag リリース：リリースノート、SemVer 推論、gh release コマンド |
| `resolving-merge-conflicts` | ハンク単位のマージコンフリクト解決：本来の意図を追跡、新たな挙動を発明しない、--abort を絶対に使わない |
| `handoff` | セッションを引き継ぎドキュメントに圧縮（パス参照、内容のコピーなし） |
| `opencode-config` | 本リポジトリの OpenCode 設定（agents/skills/commands/permissions）の作成と保守 |
| `reflect` | 継続的改善：摩擦の発見 → 最小限で保守可能な修正の提案 |
| `remove-deadcode` | 死にコードを安全に検出・削除、削除前にツールチェーン/LSP で検証 |
| `security-review` | マージ前のセキュリティレビュー（インジェクション/XSS/SSRF/シークレット/デシリアライゼーション/パストラバーサル）、報告のみで修正しない |
| `shared-language` | ドメイン用語集（CONTEXT.md）の構築、大幅なトークン節約 |
| `simplify` | 挙動を保持するコード簡素化（oracle 分析 → 適用） |
| `spec-workflow` | 軽量な規約駆動の変更：proposal → specs → design → tasks → archive |
| `verification-planning` | 実装前に最狭の検証パスを計画 |
| `verify-with-docs` | コーディング前に API ドキュメントを照合、検索優先、幻覚防止 |
| `grilling` | 要件整理インタビュー：一問ずつ、多肢選択を優先、曖昧さが収束してから着手 |
| `tech-debt-audit` | 9 次元の技術的負債監査（死にコード/重複/命名ドリフト/複雑度/依存関係/エラーハンドリング/テスト/ドキュメント/セキュリティ）、読み取り専用報告でコードは変更しない |
| `wait-what` | ユーザーメッセージが分かりにくいとき、まず一文で言い換えて確認してから着手 |
| `writing-for-agents` | エージェント向けドキュメント（skill/AGENTS.md/ポインタドキュメント）を書くためのレバレッジ |
| `to-questionnaire` | オフチャネルの一回限りアンケート（非同期記入）、リアルタイムインタビューの grilling とは別物 |
| `research` | 開かれた課題の深い調査、引用付き Markdown を生成、単点照合の verify-with-docs とは別物 |
| `wizard` | 人手によるステップバイステップウィザード（bash スクリプト、`bash -n` 検証）、人間にしかできないステップへ誘導 |

## 設計判断と反復記録

コア思想は [oh-my-openagent](https://github.com/code-yeongyu/oh-my-openagent)（意図ゲート、読み取り専用分離、アンチパターン）、[oh-my-opencode-slim](https://github.com/alvinunreal/oh-my-opencode-slim)（スケジューラ優先、フォールバックチェーン、拒否契約、プロンプトキャッシュ安全、impact×confidence÷cost）、[anomalyco/opencode](https://github.com/anomalyco/opencode)（設定スキーマ、スキル体系）、[cli/cli](https://github.com/cli/cli)（gh v2.97 コマンドセット、rate limit、gh-aw）、[OpenSpec](https://github.com/Fission-AI/OpenSpec)（delta specs、OPSX アクションフロー update/verify/四つの問い）、[mattpocock/skills](https://github.com/mattpocock/skills)（コンフリクト解決規律、引き継ぎドキュメント）、[pi](https://github.com/earendil-works/pi)（先に答えてから変更、簡潔な応答、独立セッション収集）、[deepreview](https://github.com/mechanai/deepreview)（novelty 分類の収束、有効サイズルーティング、Points of Agreement）の長所を取り入れ、純粋な設定で実現し、追加の依存関係ゼロ。

> **借用であって丸写しではない**：重すぎるパイプラインからは軽量化の設計理念だけを吸収し、冗長な機能は既存の agents/skills でカバーして新設しない。「追加より簡素化」の原則に従い、各反復でトークンの純減を目標とする。
>
> **今ラウンド（v28）の仕組みの出典**：DeepSeek キャッシュ+thinking 規律、scope-first+委任優先、原子的 TODO を AGENTS.md に集約；5 スキル新設（wait-what/writing-for-agents/to-questionnaire/research/wizard）で計 23 個；gh-cli に GHSA セキュリティ項目を 4 件追加；code-review に deepreview の自己反証を統合；.ai/calibration.yml を削除（キャリブレーション規則は code-review にインライン化）。
>
> **評価後に不採用**：mattpocock/skills のその他のプロセス系スキル（code-review、tdd、implement などは superpowers／既存スキルと重複）；superpowers には設定ノブがないため、プラグイン文字列形式での注入を維持。

### 反復マイルストーン

v1 以来 28 回の反復を重ね、継続的に上流リポジトリのベストプラクティスをベンチマークしてきました：

- **v1-v7（基礎固め）**：デュアルモデルバインド、Agent ロール体系、意図ゲートルーティング、AGENTS.md グローバルルール、Skills ディレクトリ、権限ベースライン
- **v8-v15（レビュー+規約+契約）**：code-review 二軸キャリブレーション、spec-workflow、gh-cli 整合、拒否契約、バックグラウンド検証
- **v16-v22（継続的スリム化）**：コマンド 29→18（-38%）、AGENTS.md 290→211（-27%）、一文単位の no-op 剪定、スキーマ検証による死にキー削除
- **v23-v25（整合+セキュリティ）**：上流 6 リポジトリの統合、gh-cli v2.97 エスケープインジェクション安全セクション、procedure-driven プロンプト精緻化、DCP ウィンドウ調整
- **v26（本ラウンドのスリム化）**：prune:true と tool_output 800/20480 の引き締め、DCP を 60%/30% のパーセント閾値に切り替え、writing-great-skills の代わりに grilling を導入、opencode-config を 131→64 に削減、code-review の等級化+validator、gh-cli に gh status を追加、AGENTS.md に User Override を追加、orchestrator の委任コスト規律、7 つの agent ファイルで正味 22 行削減
- **v27（削除/移行/新設）**：batch_tool の死に設定を削除、読み取り専用エージェントの無効な `write: deny`、bash の冗長 3 行を削除；Context Management セクションを orchestrator 専用サブセクションへ移行；読み取り専用エージェントの bash ホワイトリスト、read に `.env` を追加；tech-debt-audit スキルを新設；15 スキルの description を 30-40% スリム化；gh-cli に rate limit/gh skill ホスト/gh-aw など 5 点を追加、code-review に Points of Agreement を追加、spec-workflow に update の二問を追加、orchestrator に独立セッション収集+プロンプトキャッシュ安全を追加、deep-worker に impact×confidence÷cost を追加
- **v28（規律の再構築）**：キャッシュ+thinking 規律、scope-first+委任優先、原子的 TODO を AGENTS.md に集約；5 スキル新設で計 23 個；gh-cli に GHSA 4 件追加；code-review に deepreview の自己反証を統合；.ai/calibration.yml を削除（規則は code-review にインライン化）；README 十言語同期

## リポジトリ構造

```text
├── opencode/                     # OpenCode 設定ディレクトリ（独立デプロイ可能）
│   ├── agents/                   # 10 個の専任エージェント
│   │   ├── orchestrator.md       # 主入口：意図ゲート + モデル認識ルーティング
│   │   ├── planner.md            # pro：アーキテクチャと計画
│   │   ├── deep-worker.md        # pro：重量級実装
│   │   ├── oracle.md             # pro：深いコード分析（読み取り専用）
│   │   ├── reviewer.md           # pro：二軸コードレビュー（読み取り専用）
│   │   ├── consultant.md         # pro：設計案の議論と提案
│   │   ├── ui-builder.md         # pro：フロントエンドと UI
│   │   ├── explore.md            # flash：コードベース検索（読み取り専用）
│   │   ├── librarian.md          # flash：外部検索（読み取り専用）
│   │   └── light-orchestrator.md # flash：単純な編集
│   ├── skills/                   # 23 個のオンデマンドスキル
│   │   ├── code-review/          # 二軸並列レビュー + 深刻度キャリブレーション
│   │   ├── codemap/              # リポジトリ構造図の生成
│   │   ├── gh-cli/               # GitHub CLI v2.97+ リファレンス + 安全警告
│   │   ├── git-master/           # 高度な Git 操作
│   │   ├── git-release/          # Tag リリース
│   │   ├── handoff/              # セッションを引き継ぎドキュメントに圧縮
│   │   ├── opencode-config/      # メタスキル：本リポジトリの設定作成
│   │   ├── reflect/              # 継続的改善
│   │   ├── remove-deadcode/      # 死にコードの検出と削除
│   │   ├── resolving-merge-conflicts/ # ハンク単位のコンフリクト解決規律
│   │   ├── security-review/      # セキュリティレビューチェックリスト
│   │   ├── shared-language/      # ドメイン用語集（トークン節約）
│   │   ├── simplify/             # 挙動を保持するコード簡素化
│   │   ├── spec-workflow/        # 規約駆動開発
│   │   ├── tech-debt-audit/      # 技術的負債監査（9 次元、読み取り専用報告）
│   │   ├── verification-planning/ # 実装前の検証パス計画
│   │   ├── verify-with-docs/     # 検索優先の API 検証
│   │   ├── grilling/             # 要件整理インタビュー
│   │   ├── research/             # 開かれた課題の深い調査（引用付き）
│   │   ├── to-questionnaire/     # オフチャネルの一回限りアンケート
│   │   ├── wait-what/            # 分かりにくいメッセージは一文で言い換えて確認
│   │   ├── wizard/               # 人手によるステップバイステップウィザード（bash -n 検証）
│   │   └── writing-for-agents/   # エージェント向けドキュメント作成
│   ├── opencode.jsonc            # 主設定（18 コマンド）
│   ├── AGENTS.md                 # グローバルルール
│   └── dcp.jsonc                 # DCP コンテキスト圧縮（DeepSeek 128K、60%/30% パーセント閾値）
├── README.md
├── LICENSE
└── README.*.md                   # 他言語の README
```

## 利用ガイド

### モード 1：Orchestrator 自動ルーティング（デフォルト）

自然言語で要件を記述すると、Orchestrator が自動的に意図を分析し、最適なエージェントとモデルを選択して実行します。

```text
「このログイン API のエラーを調べてほしい」     → oracle が根本原因を分析 → 診断レポートを返す
「このループを最適化して、性能がひどすぎる」     → oracle が分析 → deep-worker が最適化を実装
「この PR をレビューしてほしい」               → reviewer が多次元レビュー → 等級付きレポートを返す
「ユーザーモジュールにエクスポート機能を追加したい」 → planner が計画を策定 → deep-worker が実装
「React 19 の use() API の使い方は？」        → librarian がドキュメントを調査 → シグネチャと例を返す
```

### モード 2：コマンドエイリアスで直接アクセス

| シナリオ | コマンド |
| --- | --- |
| 複雑な実装 / 複数ファイル変更 | `/deep` |
| 軽量な修正 / 単一ファイル編集 | `/quick` |
| 技術設計 / アーキテクチャ設計 | `/plan` |
| バグ調査 / 深い分析 | `/oracle` |
| コードレビュー | `/review` |
| 外部検索 / API 調査 | `/search` |
| フロントエンド / UI 作業 | `/ui` |
| 設計案の議論 / 比較と取捨選択 | `/consult` |
| 構造化デバッグ | `/oracle` |

### 典型的なワークフロー

**新機能の開発（規約駆動）：**
```text
/spec-propose  → /spec-apply  → /review
```

**バグ調査：**
```text
/oracle  → /deep  → /rmslop  → /commit
```

**コードレビュー：**
```text
/review-pr   ← PR レビュー + 自動返信
/review      ← 二軸並列レビュー
```

## 設計思想

- **純粋な設定駆動、追加の依存関係ゼロ** —— すべての能力は `opencode.jsonc` + `agents/*.md` + `skills/*/SKILL.md` + `AGENTS.md` で実現
- **DeepSeek V4 デュアルモデルの徹底活用** —— Pro は推論と意思決定、Flash は照会と軽量実行
- **トークン効率最優先** —— パス参照によるファイル貼り付けの代替、スキルのオンデマンド読み込み、圧縮の階層管理
- **プラグインは効果を高めつつ主役を奪わない** —— superpowers はプロセス規律を提供し、DCP（dcp.jsonc）は能動的な重複排除+圧縮閾値、内蔵 compaction（opencode.jsonc）は自動トリガー+prune の後始末
- **実行と探索の分離** —— deep-worker/light-orchestrator は研究・委任を禁止、explore/librarian は変更を禁止
- **キャッシュと thinking の規律** —— 静的プレフィックスを安定させ DeepSeek プロンプトキャッシュにヒットさせる；コーディングタスクは温度 0；thinking は推論タスクのみ有効化し、単純・検索タスクでは無効化
- **Scope First + Delegate Always** —— まず範囲を定め（2 ステップ以上/複数ファイル/アーキテクチャ変更はまず planner）、それから実行を委任。トップレベルのトークンはルーティングと難題だけに残す
- **原子的 TODO** —— 複数ステップのタスクはまず順序付き TODO を書き、1 件ずつ in_progress→completed；形式は `path: action for scenario — verify by check`
- **継続的改善** —— reflect が摩擦を機構的に発見、code-review の二軸キャリブレーションが品質を保証
