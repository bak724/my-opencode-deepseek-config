# My OpenCode × DeepSeek Config

[简体中文](README.md) | [繁體中文](README.zh-TW.md) | [English](README.en-US.md) | [Русский](README.ru-RU.md) | [Français](README.fr-FR.md) | [Deutsch](README.de-DE.md) | [Español](README.es-ES.md) | [Português](README.pt-BR.md) | **日本語** | [한국어](README.ko-KR.md)

**OpenCode × DeepSeek 最適構成** —— OpenCode のマルチエージェントフレームワーク上で、DeepSeek V4 デュアルモデル（Pro + Flash）の能力を最大限に引き出す構成である。中核理念：**トークン効率を最優先し、最小限のコンテキストコストで最高の開発成果を得る**。

## 現在の構成概要

- デフォルト主エージェント：`orchestrator`
- メインモデル：`deepseek/deepseek-v4-pro`、軽量モデル：`deepseek/deepseek-v4-flash`
- エージェント階層：`subagent_depth: 3`（3 階層のサブエージェントネストをサポート）
- モデル分離：`enabled_providers: ["deepseek"]` + `disabled_providers` による二重ロック
- セッション共有：無効（`share: "disabled"`）；スナップショット：有効（`snapshot: true`）
- 権限ベースライン：デフォルト許可、破壊的 bash コマンドは `ask`；`.env` 等の機密ファイルは `deny`；外部ディレクトリは `ask`
- コンテキスト圧縮：DCP 能動的圧縮（35K-75K 閾値）+ OpenCode ネイティブ compaction によるフォールバック
- グローバルルール：`AGENTS.md`（中核原則、タスク拒否契約、コンテキストとトークン効率、自己検証、アンチパターン等）
- スキル：`skills/` ディレクトリ配下の **17 個**の `SKILL.md` スキル、ネイティブ `skill` ツールでオンデマンド読み込み
- プラグイン：`superpowers`（14 個のプロセス型スキル）、`@tarquinen/opencode-dcp`（インテリジェントコンテキストトリミング）
- 実験的機能：`batch_tool` がデフォルトで有効

## DeepSeek モデル設定

### 前提条件

- OpenCode ≥ v1.14.24（DeepSeek プロバイダーは内蔵）
- DeepSeek API Key：[platform.deepseek.com/api_keys](https://platform.deepseek.com/api_keys) から申請

### 方式一：TUI 対話式設定（推奨）

```bash
opencode
# TUI 内で入力: /connect → DeepSeek を選択 → API Key を貼り付け
# 次に: /models → deepseek-v4-pro を選択
```

API Key は自動的に `~/.local/share/opencode/auth.json` へ永続化される。

### 方式二：環境変数

Windows PowerShell:
```powershell
$env:DEEPSEEK_API_KEY="sk-your-key-here"
opencode
```

恒久設定：`DEEPSEEK_API_KEY` をシステム環境変数に追加する。

### プロバイダー設定リファレンス

```jsonc
{
  "model": "deepseek/deepseek-v4-pro",
  "small_model": "deepseek/deepseek-v4-flash",
  "enabled_providers": ["deepseek"],
  "disabled_providers": ["openai", "anthropic", "google", "openrouter"]
}
```

Pro モデルで thinking/reasoning を有効にする場合、`provider` に以下を追加する：

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

> **モデル ID 命名規則**：`provider_id/model_id`、すなわち `deepseek/deepseek-v4-pro` および `deepseek/deepseek-v4-flash`。

## インストールとデプロイ

### 方式一：クローン + 環境変数（推奨、クロスプラットフォーム対応）

```bash
git clone https://github.com/znlgis/my-opencode-deepseek-config.git
```

その後、`OPENCODE_CONFIG_DIR` をリポジトリ内の `opencode/` サブディレクトリに向けるだけで使用できる。

**Windows（PowerShell）** —— 恒久設定：

```powershell
[Environment]::SetEnvironmentVariable("OPENCODE_CONFIG_DIR", "D:\path\to\my-opencode-deepseek-config\opencode", "User")
```

**Windows（PowerShell）** —— 一時設定（現在のセッションのみ）：

```powershell
$env:OPENCODE_CONFIG_DIR = "D:\path\to\my-opencode-deepseek-config\opencode"
opencode
```

**Linux / macOS** —— `~/.bashrc` または `~/.zshrc` に追記：

```bash
export OPENCODE_CONFIG_DIR="$HOME/path/to/my-opencode-deepseek-config/opencode"
```

### 方式二：シンボリックリンクでグローバル設定ディレクトリへ

**Windows（PowerShell、管理者権限が必要）：**

```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.config"
New-Item -ItemType SymbolicLink -Path "$env:USERPROFILE\.config\opencode" -Target "D:\path\to\my-opencode-deepseek-config\opencode"
```

**Linux / macOS：**

```bash
ln -s /path/to/my-opencode-deepseek-config/opencode ~/.config/opencode
```

> **互換性に関する注意**：`~/.config/opencode` は OpenCode の標準グローバル設定パスである。本リポジトリの `opencode/` サブディレクトリには `agents/`、`skills/`、`AGENTS.md` 等のファイルが含まれており、そのレイアウトは完全に OpenCode の規約に従っている。環境変数またはシンボリックリンクで指定すれば自動認識される。

### インストールの検証

OpenCode を起動して以下を確認する：
1. `/models` → 現在のモデルが `deepseek/deepseek-v4-pro` であること
2. エージェント一覧に `orchestrator`、`planner`、`deep-worker` 等 10 個のエージェントが表示されること
3. 任意のリクエストを入力し、Orchestrator が自動的に意図を分析してルーティングすること

## モデル分業

本リポジトリは DeepSeek V4 デュアルモデル内での分業に厳格に限定し、他のモデルを導入しない：

| モデル | 用途 |
| --- | --- |
| `deepseek/deepseek-v4-pro` | 計画、アーキテクチャ、根本原因解析、コードレビュー、重量級実装、マスタースケジューリング |
| `deepseek/deepseek-v4-flash` | 高速探索、外部検索、軽量タスク、単純編集 |

### ルーティング戦略

- **Flash 優先**：検索、ルックアップ、単純編集等、明確に定義されたタスクは Flash エージェントを優先
- **Pro は推論に集中**：計画、解析、レビュー、複雑な実装——Pro のみ使用
- **自動昇格**：Flash エージェントが処理不能な場合、自動的に Pro へ昇格（完全なコンテキスト付き）

## エージェント構造

### プライマリエージェント

| エージェント | モデル | 役割 |
| --- | --- | --- |
| `orchestrator` | v4-pro | デフォルトエントリポイント：意図ゲーティング（Intent Gate）+ モデル認識ルーティング + フォールバックチェーン |

### サブエージェント

| エージェント | モデル | 権限 | 役割 |
| --- | --- | --- | --- |
| `planner` | v4-pro | 読み書き | 計画、アーキテクチャ、タスク分割 |
| `deep-worker` | v4-pro | 読み書き | 重量級実装、マルチファイル変更、複雑なデバッグ |
| `oracle` | v4-pro | **読み取り専用** | 根本原因解析、コードの深層理解 |
| `reviewer` | v4-pro | **読み取り専用** | 二軸コードレビュー（規約 + 仕様）+ 重大度キャリブレーション |
| `ui-builder` | v4-pro | 読み書き | フロントエンド・UI 関連タスク |
| `consultant` | v4-pro | 読み書き | 方式検討、ベストプラクティス提案 |
| `explore` | v4-flash | **読み取り専用** | コードベース検索、並列探索 |
| `librarian` | v4-flash | **読み取り専用** | ドキュメント検索、Web 検索 |
| `light-orchestrator` | v4-flash | 読み書き | 軽量タスク、単一ファイル編集 |

> `deep-worker` と `light-orchestrator` は「研究禁止・委任禁止」の原則に従う——実行のみ、探索は行わず、コンテキストは orchestrator が提供する。

## ショートカットコマンド

### エージェントルーティングコマンド

| コマンド | エージェント | 用途 |
| --- | --- | --- |
| `/deep` | `deep-worker` | 重量級実装、マルチファイル変更 |
| `/quick` | `light-orchestrator` | 軽量タスク、単一ファイル編集 |
| `/ui` | `ui-builder` | フロントエンド/UI 作業 |
| `/review` | `reviewer`（code-review） | 二軸並行レビュー（規約+仕様）+ 重大度キャリブレーション |
| `/review-pr` | `reviewer`（code-review + gh-cli） | PR レビューと GitHub への返信投稿 |
| `/plan` | `planner` | 計画立案、技術方式 |
| `/search` | `librarian` | 外部検索、ドキュメント参照 |
| `/oracle` | `oracle` | 深層解析、問題の原因特定 |
| `/consult` | `consultant` | 相談、比較、提案 |

### 操作コマンド

| コマンド | エージェント | 用途 |
| --- | --- | --- |
| `/commit` | `light-orchestrator` | Conventional Commits コミットメッセージ生成（インライン形式） |
| `/release` | `deep-worker`（git-release） | タグリリース準備 |
| `/reflect` | `oracle`（reflect） | 摩擦の発見 → 設定最適化の提案 |
| `/handoff` | `light-orchestrator`（handoff） | セッションを引継ぎドキュメントに圧縮 |

### インラインコマンド

| コマンド | エージェント | 用途 |
| --- | --- | --- |
| `/codemap` | `explore`（codemap） | リポジトリ構造図の生成 |
| `/simplify` | `oracle`（simplify）→ `light-orchestrator` | oracle の解析 → light-orchestrator が簡略化を適用 |
| `/rmslop` | `deep-worker`（remove-deadcode） | デッドコードと AI slop のクリーンアップ |

### 仕様駆動コマンド

| コマンド | エージェント | 用途 |
| --- | --- | --- |
| `/spec-propose` | `planner`（spec-workflow） | コード探索 → 変更提案の起草 |
| `/spec-apply` | `deep-worker`（spec-workflow） | tasks.md に従い逐次実装 → 自動アーカイブ |

## スキル（Skills）

OpenCode はネイティブ `skill` ツールを通じてスキルをオンデマンドで公開する——エージェントは必要な時だけ読み込み、コンテキストを占有しない。

| スキル | 役割 |
| --- | --- |
| `code-review` | 二軸並行レビュー（規約 + 仕様）+ 重大度キャリブレーション |
| `codemap` | 注釈付きリポジトリ構造図の生成、探索トークンの節約 |
| `gh-cli` | GitHub CLI v2.97+ 完全リファレンス（Issues 2.0、copilot、agent-task、gh skill） |
| `git-master` | 高度な Git 操作：rebase、squash、bisect、reflog、worktree |
| `git-release` | タグリリース：SemVer 推論、リリースノート、gh release コマンド |
| `resolving-merge-conflicts` | コンフリクトをhunkごとに解決：元の意図を追跡し、新しい動作を発明せず、--abortは絶対に使用しない |
| `handoff` | セッションを引継ぎドキュメントに圧縮（パス参照、内容コピーなし） |
| `opencode-config` | OpenCode 設定の作成と保守 |
| `reflect` | 継続的改善：摩擦の発見 → 最小限の修正を提案 |
| `remove-deadcode` | デッドコードの安全な検索と削除、削除前に LSP 検証 |
| `security-review` | マージ前の diff に対するセキュリティレビュー |
| `shared-language` | ドメイン用語集の構築、コンテキストトークンの大幅節約 |
| `simplify` | 振る舞いを保持したコード簡略化（oracle 解析 → light-orchestrator 適用） |
| `spec-workflow` | 軽量仕様駆動変更（propose → design → tasks → implement → archive） |
| `verification-planning` | 実装前に最も狭い検証パスを計画 |
| `verify-with-docs` | コーディング前に API ドキュメントを検証、検索優先、幻覚防止 |
| `writing-great-skills` | スキル作成規約：no-op トリミング、肯定的表現、完了基準 |

## 設計判断とイテレーション記録

中核となる考え方は [oh-my-openagent](https://github.com/code-yeongyu/oh-my-openagent)（意図ゲーティング、読み取り専用分離、アンチパターン）、[oh-my-opencode-slim](https://github.com/alvinunreal/oh-my-opencode-slim)（スケジューラー優先、フォールバックチェーン、拒否契約）、[anomalyco/opencode](https://github.com/anomalyco/opencode)（設定スキーマ、スキル体系）、[cli/cli](https://github.com/cli/cli)（gh v2.97 完全コマンドセット）、[OpenSpec](https://github.com/Fission-AI/OpenSpec)（delta specs、変更提案更新）、[mattpocock/skills](https://github.com/mattpocock/skills)（conflict resolution discipline, handoff docs）、[pi](https://github.com/earendil-works/pi)（先に回答し後から編集、簡潔な応答）、[deepreview](https://github.com/mechanai/deepreview)（novelty-based convergence, effective-size routing）の長所を参考にし、純粋な設定で実現、追加依存ゼロである。

> **参考であって模倣ではない**：過重なパイプラインからは軽量設計理念のみを抽出。冗長機能は既存の agents/skills でカバーし、新規追加は行わない。「削減を追加より優先」の原則に従い、各イテレーションでトークンの純減を目標とする。

### イテレーションマイルストーン

| 段階 | 主要変更 |
| --- | --- |
| **v1-v7（基盤構築）** | デュアルモデルバインディング、エージェントロール体系、意図ゲーティング/分類ルーティング、AGENTS.md グローバルルール、skills ディレクトリとコマンドエイリアス、権限ベースライン |
| **v8-v12（レビュー+仕様）** | code-review 強化（段階分け/自己チェック/拒否基準）、spec-workflow 確立（explore→propose→apply→archive）、deepwork/reflect/verification-planning 追加、gh-cli を v2.96+ にアライン |
| **v13-v15（契約+削減）** | AGENTS.md に Evidence Discipline / Task Rejection Contract / stop condition 追加。エージェントプロンプトとグローバルルールの完全重複排除。バックグラウンドサブエージェントエラーチェック追加 |
| **v16-v18（効率的実行）** | 神話的名称の除去、ルーティングテーブル統合、gh-cli を Issues 2.0 に拡張、spec-workflow に verify + 判断フレームワーク追加 |
| **v19（上流との整合）** | 6 つの上流リポジトリを再検証。`/review-pr` の行単位コメントバグ修正。code-review ルーティングを生の行数から実効ロジックボリュームに変更 |
| **v20（リファクタリング最適化）** | `agent/`→`agents/` に OpenCode 推奨へ統一。AGENTS.md 22%削減（290→227 行）。`diagnose`（6 段階デバッグ）+ `handoff`（セッション引継ぎ）スキル追加。spec-workflow に `/update` 追加。code-review にエントロピースキャン+収束チェック追加。エージェントプロンプト 20%重複排除 |
| **v21（全面的スリム化リファクタリング）** | スキル 18→17（deepwork/conventional-commits/diagnose 削除、writing-great-skills/shared-language 追加）。コマンド 29→18（-38%）。AGENTS.md 227→212 行（-7%）。スキルを一文単位で no-op トリミング。code-review の二軸並行+キャリブレーションファイル機構。pi/deepreview/mattpocock 等 6 リポジトリの実戦経験を参考 |
| **v22（スキーマ検証とスリム化）** | OpenCode と DCP 公式スキーマを検証：無効な `agent.fallback` デッドキーを削除。`dcp.jsonc` の全キーが合法であることを確認（v3.1.14）、ゼロ変更で盲目的な追加なし。AGENTS.md の「トークン効率」を「コンテキスト管理」に統合し完全重複排除、`Self-Verification` の宙吊り参照を修正（212→197 行）。orchestrator の 3 つのルーティングテーブルを統合（128→79 行、-38%、Intent Gate/Agent Directory/Fallback は完全保持）。14 スキルからパーサーに無視される `license/compatibility/metadata` frontmatter を除去（-70 行）。`tool_output` を能動的トークン節約にダウンサイズ（1500 行/40KB） |
| **v23（二重規律統合+削減統合）** | 6 つの上流リポジトリに基づく最終統合：`gh-skill` 削除（機能を `gh-cli` Agent Skills セクションに統合、-122 行）。`verification-planning` のデッドリファレンス修正。AGENTS.md に Pi 由来の二重規律追加（先に回答し後から編集+表明、グローバル簡潔性）+ slim 委任契約+ジョブボード + deepreview ファイル IPC（+15 行）。orchestrator テーブルのモデル列重複排除（Pro/Flashサブテーブルに再構成、79→86）。reviewer に検証者デフォルト拒否スタンスを補完（+6 行）。gh-cli Agent Skills セクション強化（+10 行）。純減 ~90 行、スキル 17→16 |
| **v24（Upstream v2.97 + Conflict Resolution）** | gh-cli fully upgraded to v2.97.0（fixed 11 outdated items）；added `project` by-name editing, `issue develop`, `org list` commands；code-review entropy scan renamed to Mechanical scan, convergence enhanced with converging/deadlocked/diverging triage；added `resolving-merge-conflicts`；calibration.yml now has 3 active downgrade rules. skills 16→17. |

## リポジトリ構造

```text
├── opencode/                     # OpenCode 設定ディレクトリ（独立デプロイ可能）
│   ├── .ai/
│   │   └── calibration.yml       # code-review 重大度キャリブレーション
│   ├── agents/                   # 10 個の専任エージェント
│   │   ├── orchestrator.md       # メインエントリ：意図ゲーティング + モデル認識ルーティング
│   │   ├── planner.md            # pro：アーキテクチャと計画
│   │   ├── deep-worker.md        # pro：重量級実装
│   │   ├── oracle.md             # pro：深層コード解析（読み取り専用）
│   │   ├── reviewer.md           # pro：二軸コードレビュー（読み取り専用）
│   │   ├── consultant.md         # pro：方式検討と提案
│   │   ├── ui-builder.md         # pro：フロントエンドと UI
│   │   ├── explore.md            # flash：コードベース検索（読み取り専用）
│   │   ├── librarian.md          # flash：外部検索（読み取り専用）
│   │   └── light-orchestrator.md # flash：単純編集
│   ├── skills/                   # 17 個のオンデマンドスキル
│   │   ├── code-review/          # 二軸並行レビュー + 重大度キャリブレーション
│   │   ├── codemap/              # リポジトリ構造図の生成
│   │   ├── gh-cli/               # GitHub CLI v2.97+ リファレンス
│   │   ├── git-master/           # 高度な Git 操作
│   │   ├── git-release/          # タグリリース
│   │   ├── resolving-merge-conflicts/ # コンフリクト解決規律
│   │   ├── handoff/              # セッションを引継ぎドキュメントに圧縮
│   │   ├── opencode-config/      # メタスキル：本リポジトリ設定の作成
│   │   ├── reflect/              # 継続的改善
│   │   ├── remove-deadcode/      # デッドコード検出と削除
│   │   ├── security-review/      # セキュリティレビューチェックリスト
│   │   ├── shared-language/      # ドメイン用語集（トークン節約）
│   │   ├── simplify/             # 振る舞いを保持したコード簡略化
│   │   ├── spec-workflow/        # 仕様駆動開発
│   │   ├── verification-planning/ # 実装前検証パス計画
│   │   ├── verify-with-docs/     # 検索優先 API 検証
│   │   └── writing-great-skills/ # スキル作成規約
│   ├── opencode.jsonc            # メイン設定（18 コマンド）
│   ├── AGENTS.md                 # グローバルルール（~212 行）
│   └── dcp.jsonc                 # DCP コンテキスト圧縮（DeepSeek 128K）
├── README.md
├── LICENSE
└── README.*.md
```

## 利用ガイド

### モード一：Orchestrator 自動ルーティング（デフォルト）

自然言語で要件を記述すると、Orchestrator が自動的に意図を分析し、最適なエージェントとモデルを選択して実行する。

```text
「このログイン API のエラーを調査して」        → oracle が根本原因を解析 → 診断レポートを返却
「このループ、パフォーマンスが悪すぎるから最適化して」 → oracle が解析 → deep-worker が最適化を実施
「この PR をレビューして」                      → reviewer が多面的にレビュー → 段階別レポートを返却
「ユーザーモジュールにエクスポート機能を追加したい」   → planner が方式を策定 → deep-worker が実装
「React 19 の use() API の使い方は？」          → librarian がドキュメントを検索 → シグネチャと例を返却
```

### モード二：コマンドエイリアス直接指定

| シナリオ | コマンド |
| --- | --- |
| 複雑な実装 / マルチファイル変更 | `/deep` |
| 軽量な変更 / 単一ファイル編集 | `/quick` |
| 技術方式の策定 / アーキテクチャ設計 | `/plan` |
| バグ調査 / 深層解析 | `/oracle` |
| コードレビュー | `/review` |
| 外部検索 / API 検索 | `/search` |
| フロントエンド / UI 作業 | `/ui` |
| 方式検討 / 比較とトレードオフ | `/consult` |
| 構造化デバッグ | `/oracle` |

### 典型的なワークフロー

**新機能開発（仕様駆動）：**
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
/review      ← 二軸並行レビュー
```

## 設計哲学

- **純粋設定駆動、追加依存ゼロ** —— 全機能は `opencode.jsonc` + `agents/*.md` + `skills/*/SKILL.md` + `AGENTS.md` で実現
- **DeepSeek V4 デュアルモデルを極限まで活用** —— Pro は推論と意思決定、Flash は検索と軽量実行
- **トークン効率を最優先** —— パス参照でファイル貼り付けを代替、スキルはオンデマンド読み込み、圧縮は階層管理
- **プラグインは增效するが主役を奪わない** —— superpowers はプロセス規律を提供、DCP はインテリジェント圧縮で単純な切り捨てを代替
- **実行と探索の分離** —— deep-worker/light-orchestrator は研究/委任禁止、explore/librarian は変更禁止
- **継続的改善** —— reflect による摩擦の機構的発見、code-review の二軸キャリブレーションが品質を保証
