---
title: "oh-my-opencodeのClaude Code版を作った"
emoji: "🪨"
type: "tech"
topics: ["claudecode", "ai", "cli", "plugin"]
published: true
---

## はじめに

> 業務自動化Pythonエンジニア。バイブコーディング歴1年 ≒ エンジニア歴。

### きっかけ

AnthropicがサードパーティツールのOAuth認証をブロックした。

oh-my-opencodeという便利なプラグインがあったが、いつまた動かなくなるかわからない。じゃあClaude Code向けに自作してみよう。

## 作ったもの：o-m-cc

**Sisyphus Loop for Claude Code** - TODOが完了するまで止まらないマルチエージェントワークフロー

oh-my-opencodeの考え方を真似した。エージェントの名前とかはそのまま使わず、一番コアな思想・仕組みの名前（Sisyphus）だけあえてそのままにした。

SDDみたいにSisyphusが一つの流行りワードになったりして。怒られたら変える。

### Sisyphusとは

ギリシャ神話のシーシュポス。永遠に岩を山頂に押し上げ続ける男。

このプラグインは「タスク完了まで決して止まらない」というマインドセットをClaude Codeに注入する。

具体的には ralph-wiggum プラグインと連携して、`<promise>DONE</promise>` が出力されるまでループし続ける。途中で止まらない。

### 仕様駆動開発（SDD）

cc-sddを参考に、いきなりコードを書かせない設計にした。複雑なタスクは計画から：

```
要件定義 → ギャップ分析 → 設計 → タスク分解 → 実装
```

計画フェーズをしっかり踏んでから実装に入るので、手戻りが少ない。

簡単なタスクも、標準のtodoツールで簡単にリスト化してSisyphusモードで作業してくれる。

## コマンド

### セットアップ

```bash
# プロジェクト初期化（CLAUDE.md作成 + Sisyphus有効化）
/o-m-cc:init
```

既存プロジェクト（CLAUDE.md あり）でも `/o-m-cc:init` でOK。Sisyphusセクションのみ追加される。

実行すると：
- CLAUDE.md に Sisyphus 原則を追加
- hooks 設定（`<promise>DONE</promise>` までループ）
- 依存プラグインの確認・インストール
- LSP機能（pyright-lsp、typescript-lsp）もインストール

LSPはOpenCodeを使っていてよかった部分。Claude Codeでも最近追加できるようになった。実装中に早めにエラーに気づけるので、コードレビューで一気にチェックする前にちょこちょこ直していける。

### 計画フェーズ

| コマンド | 説明 | Context |
|---------|------|---------|
| `/o-m-cc:requirements <task>` | 要件定義 | - |
| `/o-m-cc:design` | 設計書作成 | - |
| `/o-m-cc:tasks` | タスク分解 | - |
| `/o-m-cc:plan <task>` | 上記を一括実行（scout によるギャップ分析含む） | fork |

AskQuestionでサジェストが出るようになっているので、次のコマンドを選びやすい。

### 実行

| コマンド | 説明 | Context |
|---------|------|---------|
| `/o-m-cc:ultrawork <task>` | 並列エージェントで最大パフォーマンス実行（自動 /compact） | fork |

> **Context: fork** - サブエージェント実行時のコンテキスト汚染を防止。探索結果やレビュー詳細がメイン会話を汚さない。

ultrawork 開始時に自動的に `/compact` を実行。プランファイル（`.plan/`）にすべての情報が保存されているため、会話履歴のクリーンアップが安全に行える。

実行するとこんな感じ：

```
╔══════════════════════════════════╗
║  ⚡ ULTRAWORK MODE ENABLED ⚡    ║
║  Parallel Agent Orchestration    ║
╚══════════════════════════════════╝
```

完了時：

```
🚀 ULTRAWORK COMPLETE

実行サマリー:
- 起動エージェント: X個
- 並列タスク: X個
- 完了タスク: X/X

レビュー結果:
- Critical: なし
- Warning: X件

✅ 全要件達成
```

（まだ2回くらいしか試してないので調整入るかも）

### 品質

| コマンド | 説明 | Context |
|---------|------|---------|
| `/o-m-cc:review [files]` | コードレビュー（security-guidance連携 + code-simplifier提案） | fork |

oh-my-opencodeだとDONEでそのまま終了だが、o-m-ccは実装完了後にコードレビューを挟む：

```
実装完了 → @code-reviewer（並列でセキュリティチェック等） → 問題あり？ → 修正ループへ
```

security-guidance や code-simplifier と連携して、問題があれば再度修正ループに突入する。

## ワークフロー

### /o-m-cc:plan の流れ

```
┌──────────────┐    ┌──────────────┐
│  Phase 1     │───▶│  Phase 1.5   │
│  要件定義    │    │  ギャップ    │
│  (analyst)   │    │  (scout)     │
└──────────────┘    └──────────────┘
        ↓
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  Phase 2     │───▶│  Phase 3     │───▶│  Phase 4     │
│  設計        │    │  タスク分解  │    │  レビュー    │
│  (designer)  │    │  (planner)   │    │  (critic)    │
└──────────────┘    └──────────────┘    └──────────────┘
```

**Phase 1.5 (scout)** はcc-sddを参考にo-m-ccで追加した部分：
- @scout エージェントは**必ず質問で終わる**ように設計
- 「何か質問ありますか？」ではなく、具体的な質問を投げかけて終わる
- 要件の漏れ・曖昧さを発見
- Criticalな質問が解決するまで続行

## サブエージェント

13個のエージェントを用途別に使い分け：

### 計画系
| Agent | 役割 | Permission |
|-------|------|------------|
| @analyst | 現状分析・要件定義 | write |
| @scout | ギャップ分析（必ず質問で終わる） | plan |
| @designer | アーキテクチャ設計 | write |
| @planner | タスク分解 | write |
| @critic | 計画レビュー | plan |

### 分析系
| Agent | 役割 | Permission |
|-------|------|------------|
| @advisor | デバッグ・戦略相談 | plan |
| @researcher | ドキュメント調査（深度自動判断） | plan |
| @explore | 高速コード探索 | plan |
| @vision | PDF/画像分析 | plan |

### 実装・品質系
| Agent | 役割 | Permission |
|-------|------|------------|
| @frontend | UI/UXコンポーネント作成 | write |
| @document-writer | ドキュメント作成 | write |
| @code-reviewer | コードレビュー | default |
| @security-reviewer | セキュリティレビュー | default |

> `@code-reviewer` と `@security-reviewer` は**並列実行推奨**

> **Permission**:
> - `plan`: 読み取り専用モード。権限確認なしで高速動作
> - `write`: 書き込み可能（Write/Edit ツール使用）
> - `default`: Bashなど特殊ツール使用のため標準権限

## 依存プラグイン

`/o-m-cc:init` 実行時に以下のプラグインを確認・インストール：

| プラグイン | 用途 |
|-----------|------|
| frontend-design | フロントエンド設計支援 |
| feature-dev | 機能開発ワークフロー |
| code-simplifier | コード簡素化（レビュー後提案） |
| security-guidance | セキュリティレビュー支援 |

> **Note**: ループ制御（`<promise>DONE</promise>` 検知）は o-m-cc 内蔵の Stop Hook で実現。外部プラグイン不要。

## 使ってみた感想

### 正しくサブエージェントに振り分けてくれる

タスクの内容に応じて適切なエージェントを選んでくれる。設計は @designer、コード探索は @explore、レビューは @code-reviewer といった具合。

### タスク分解がしっかりしている

計画フェーズでタスクがしっかり分解される：

```
Phase 1: 設定（並列可能）
┌──────┬───────────────────────────────────────────┬──────┐
│ Task │                   説明                    │ 見積 │
├──────┼───────────────────────────────────────────┼──────┤
│ 1    │ Archive.js - archiveImageGenSheet() 追加  │ M    │
├──────┼───────────────────────────────────────────┼──────┤
│ 2    │ 225_staging Config - IMAGE_GEN_SHEET_NAME │ S    │
└──────┴───────────────────────────────────────────┴──────┘
```

Phase分け、並列可否、見積もり（S/M/L）まで出してくれる。

### 手離れが良い

一度計画を承認したら、あとは「実装開始して」または `/o-m-cc:ultrawork` と言うだけ。TODO完了まで止まらずに進めてくれるので、他の作業をしながら待てる。

## インストール

```bash
# 1. marketplace 追加
claude plugin marketplace add kok1eee/o-m-cc

# 2. プラグインインストール
claude plugin install o-m-cc@kok1eee

# 3. プロジェクト初期化
/o-m-cc:init
```

## Orchestration（構造化実行）

`/o-m-cc:tasks` で tasks.md と同時に `orchestration.yml` が生成される。
`/o-m-cc:ultrawork` はこのファイルを読み込み、構造化された実行を行う。

```yaml
task_groups:
  - name: "Phase 1: 基盤構築"
    agent: "general-purpose"
    tasks: ["TASK-001"]

  - name: "Phase 2: 機能実装"
    agent: "frontend"
    tasks: ["TASK-002", "TASK-003"]

parallel_groups:
  - ["Phase 1", "Phase 2"]
```

| モード | 条件 | 動作 |
|--------|------|------|
| **Orchestrated** | `orchestration.yml` あり | YML定義に従って構造化実行 |
| **Free** | `orchestration.yml` なし | 従来の自由形式で実行 |

## Standards & Steering

プロジェクト固有の規約とコンテキストを `.claude/` 配下で管理。

### Standards（技術規約）

実装時にエージェントが参照する技術規約：

```
.claude/standards/
├── global/          # コーディングスタイル、規約、技術スタック
├── frontend/        # フロントエンド規約
├── backend/         # API設計規約
├── testing/         # テスト戦略
└── learned/         # 発見したパターン（動的）
```

### Steering（プロジェクト文脈）

計画時にエージェントが参照するプロジェクト文脈：

```
.claude/steering/
├── product.md     # プロダクト概要、目的
├── tech.md        # アーキテクチャ、技術選定理由
└── structure.md   # ディレクトリ構造
```

| | Standards | Steering |
|---|-----------|----------|
| 目的 | 実装品質の統一 | 計画の文脈提供 |
| 参照タイミング | 実装時 | 計画時 |
| 内容 | How（どう実装するか） | What/Why（何を、なぜ） |

## Token Efficiency

エージェントの出力は **要約 + ログ分離** でトークン消費を抑制。

```
サブエージェント
    ├─ 詳細 → .plan/logs/{agent}-{timestamp}.md に保存
    └─ 要約 → メインエージェント（Sisyphus）に返却
```

初期読み込みコスト：約910トークン（Opus 200k の約 **0.5%**）

## Cross-Session Restoration

セッション開始時に前回の状態を自動表示。`/compact` 後も継続作業可能。

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📋 Previous Session Found (.plan/handoff.yaml)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Status: in_progress
Current Task: TASK-003 (UserService の実装)
Next Steps: UserService のテスト追加

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
💡 To continue: describe what you want to work on
💡 To start fresh: /clear
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

ultrawork 完了時に `.plan/handoff.yaml` が自動生成される。

## Research Depth Levels

`@researcher` エージェントは、リクエストのキーワードから調査深度を自動判断。

| 深度 | トリガーキーワード |
|------|------------------|
| **quick** | ざっくり、簡単に、概要 |
| **standard** | （デフォルト） |
| **deep** | 詳しく、比較して、深掘り |
| **exhaustive** | 徹底的に、網羅的に、全部 |

```
「React Hooks の使い方をざっくり教えて」     → quick
「JWT認証の実装方法を調べて」               → standard
「Prisma vs TypeORM を詳しく比較して」      → deep
```

## 今後の展望

現状でも @advisor や @code-reviewer などの専門家はいる。

ただ、より狭い範囲の専門家を追加してもいいかも：
- Python専門家（python-master skill連携）
- GAS専門家（gas-master skill連携）

プラグインの構造上、skillとの連携が自然にできる。

## まとめ

- oh-my-opencodeが使えなくなるかもしれないので自作
- Sisyphus哲学でTODO完了まで止まらない
- @scoutの必ず質問で終わる設計で要件の漏れを発見
- ultraworkで並列エージェント実行（自動/compact）
- orchestration.ymlで構造化実行
- Standards & Steeringでプロジェクト規約管理
- Cross-Session Restorationでセッション復元

GitHub: [kok1eee/o-m-cc](https://github.com/kok1eee/o-m-cc)

## 参考

- [cc-sdd](https://github.com/gotalab/cc-sdd) - 仕様駆動開発
- [oh-my-opencode](https://github.com/code-yeongyu/oh-my-opencode) - インスパイア元
- [ralph-wiggum](https://ghuntley.com/ralph/) - Sisyphus哲学
