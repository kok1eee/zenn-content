---
title: "o-m-cc: Sisyphus Loop で Claude Code を止まらない開発マシンに"
emoji: "🪨"
type: "tech"
topics: ["claudecode", "ai", "cli", "plugin", "automation"]
published: true
---

## はじめに

> 業務自動化Pythonエンジニア。バイブコーディング歴1年 ≒ エンジニア歴。

Claude Code 用のプラグイン **o-m-cc** を作った。

oh-my-opencode という便利なプラグインがあったが、Anthropic がサードパーティツールの OAuth をブロックしたりと、いつまた動かなくなるかわからない。じゃあ Claude Code 向けに自作しようと思った。

## 作ったもの

**Sisyphus Loop for Claude Code** - TODO が完了するまで止まらないマルチエージェントワークフロー

GitHub: [kok1eee/o-m-cc](https://github.com/kok1eee/o-m-cc)

## Sisyphus 哲学

ギリシャ神話のシーシュポス。永遠に岩を山頂に押し上げ続ける男。

このプラグインは「タスク完了まで決して止まらない」というマインドセットを Claude Code に注入する。

### 原則

| 原則 | 説明 |
|------|------|
| **Iteration > Perfection** | 最初から完璧を目指さない。ループで改善 |
| **Failures Are Data** | 失敗は情報。学んでプロンプトを調整 |
| **Persistence Wins** | 成功するまで試し続ける |

## 主な機能

### 1. 仕様駆動開発（SDD）

いきなりコードを書かせない。複雑なタスクは計画から：

```
要件定義 → ギャップ分析 → 設計 → タスク分解 → 実装
```

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  Phase 1     │───▶│  Phase 1.5   │───▶│  Phase 2     │───▶│  Phase 3     │───▶│  Phase 4     │
│  要件定義    │    │  ギャップ    │    │  設計        │    │  タスク分解  │    │  レビュー    │
│  (analyst)   │    │  (scout)     │    │  (designer)  │    │  (planner)   │    │  (critic)    │
└──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
```

**Phase 1.5 (scout)** がポイント：
- Prometheus 式インタビュー
- **必ず質問で終わる**（パッシブ終了禁止）
- 要件の漏れ・曖昧さを発見

### 2. 15 のエージェント

専門家チームを用途別に使い分け：

#### 計画系

| Agent | 役割 |
|-------|------|
| @analyst | 現状分析・要件定義 |
| @scout | ギャップ分析（必ず質問で終わる） |
| @designer | アーキテクチャ設計 |
| @planner | タスク分解 |
| @critic | 計画レビュー |

#### 分析系

| Agent | 役割 |
|-------|------|
| @advisor | デバッグ・戦略相談 |
| @researcher | ドキュメント調査（深度自動判断） |
| @explore | 高速コード探索 |
| @vision | PDF/画像分析 |

#### 実装・品質系

| Agent | 役割 |
|-------|------|
| @frontend | UI/UX コンポーネント作成 |
| @document-writer | ドキュメント作成 |
| @code-reviewer | コードレビュー |
| @security-reviewer | セキュリティレビュー |
| @code-simplifier | コード簡素化 |

### 3. Hooks システム

Claude Code の hooks 機能を活用した自動化。8 つのフックで開発体験を向上。

#### 概要

| タイミング | Hook | 説明 |
|-----------|------|------|
| SessionStart | `check-dependencies.sh` | 依存コマンド確認 |
| SessionStart | `archive-plans.sh` | 古いプランをアーカイブ |
| SessionStart | `resume-session.sh` | 前回セッション復元 |
| Stop | `stop-guard.sh` | Sisyphus ガード |
| Stop | `generate-handoff.sh` | ハンドオフ生成 |
| UserPromptSubmit | `focus-guard.sh` | 脱線防止 |
| PreToolUse | `security_reminder_hook.py` | セキュリティ警告 |
| PostToolUse | `auto-verify.sh` | 自動検証 |

#### フォーカスガード（focus-guard.sh）

タスク進行中に別の話をしても、脱線せずに作業を続ける。

```
📋 タスク進行中 (残り 5 件: Phase 2: Implementation)

作業中の割り込み対応:
- 現在の作業に関連する修正・方向転換 → 反映する
- 全く別の作業の依頼 → 「現在のタスク完了後に対応します」と返答
```

- 強制ブロックではなく systemMessage 注入
- Claude が柔軟に判断（関連する修正は受け入れる）
- 完全に別の話なら後回しにする

#### Sisyphus ガード（stop-guard.sh）

`<promise>DONE</promise>` を検知した時：

1. code-reviewer が実行されたか確認
2. Critical な問題があればループ継続
3. 問題なければ終了許可

```
════════════════════════════════════════════════════════
  🔴 SISYPHUS GUARD: Critical な問題が検出されました
════════════════════════════════════════════════════════

  code-reviewer が Critical な問題を指摘しています。
  修正してから再度レビューを実行してください。
```

#### セッション復元（resume-session.sh + generate-handoff.sh）

セッション終了時に状態を `handoff.yaml` に保存。次回開始時に自動復元。

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📋 Previous Session Found (spec/plan/handoff.yaml)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Status: in_progress
Current Task: Phase 2 - UserService の実装

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
💡 To continue: describe what you want to work on
💡 To start fresh: /clear
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

`/compact` 後も作業を継続できる。

## 使い方

### インストール

```bash
# 1. marketplace 追加
claude plugin marketplace add kok1eee/o-m-cc

# 2. プラグインインストール
claude plugin install o-m-cc@kok1eee

# 3. プロジェクト初期化
/o-m-cc:init
```

### 簡単なタスク

普通に依頼するだけ：

```
「APIのエラーハンドリングを追加して」
「ボタンの色を青に変更して」
```

自動的に TODO 作成 → 実装 → レビュー → 完了。

### 複雑なタスク

計画フェーズを先に：

```bash
# 一括実行
/o-m-cc:plan "認証システムを実装"

# または段階的に
/o-m-cc:requirements "認証システムを実装"
/o-m-cc:design
/o-m-cc:tasks
```

計画完了後、実装を依頼：

```
「計画に沿って実装を開始して」
```

### 最大パフォーマンス

```bash
/o-m-cc:ultrawork "認証機能を実装"
```

並列エージェントで最速実行。開始時に自動 `/compact`。

## コマンド一覧

| コマンド | 説明 |
|---------|------|
| `/o-m-cc:init` | プロジェクト初期化 |
| `/o-m-cc:requirements <task>` | 要件定義 |
| `/o-m-cc:design` | 設計書作成 |
| `/o-m-cc:tasks` | タスク分解 |
| `/o-m-cc:plan <task>` | 上記を一括実行 |
| `/o-m-cc:review [files]` | コードレビュー |
| `/o-m-cc:ultrawork <task>` | 並列実行 |

## トークン効率

初期読み込みコスト：約 910 トークン（Opus 200k の約 **0.5%**）

エージェントの出力は **要約 + ログ分離** でトークン消費を抑制：

```
サブエージェント
    ├─ 詳細 → spec/plan/logs/{agent}-{timestamp}.md に保存
    └─ 要約 → メインエージェントに返却
```

## 向いているタスク

### ✅ 向いている

- **大規模リファクタリング** - フレームワーク移行
- **バッチ処理** - ドキュメント生成、コード標準化
- **テストカバレッジ向上** - 全関数にテスト追加
- **新規プロジェクト** - 足場作り

### ❌ 向いていない

- **人間の判断が必要** - デザイン決定、UX 評価
- **一発で終わる操作** - ファイルコピー
- **成功基準が曖昧** - 「いい感じに」

## まとめ

- **Sisyphus 哲学**: TODO 完了まで止まらない
- **SDD フロー**: 要件 → 設計 → タスク → 実装
- **Prometheus 式インタビュー**: @scout が必ず質問で要件の漏れを発見
- **8 つの Hooks**: 脱線防止、セッション復元、自動レビュー
- **15 のエージェント**: 専門家チーム

GitHub: [kok1eee/o-m-cc](https://github.com/kok1eee/o-m-cc)

## 参考

- [cc-sdd](https://github.com/gotalab/cc-sdd) - 仕様駆動開発
- [oh-my-opencode](https://github.com/code-yeongyu/oh-my-opencode) - インスパイア元
