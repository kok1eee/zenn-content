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
# Sisyphus モードを有効化
/o-m-cc:sisyphus
```

インストーラーではなくpluginとして配布したかったので、セットアップ用のコマンドを用意した。

Claudeが一緒に現在の設定を見ながらやってくれるので、問答無用で上書きせず必要な部分だけ修正してくれる。実行すると：
- CLAUDE.md に Sisyphus 原則を追加
- hooks 設定（`<promise>DONE</promise>` までループ）
- 依存プラグインの確認・インストール
- LSP機能（pyright-lsp、typescript-lsp）もインストール

### 計画フェーズ

| コマンド | 説明 |
|---------|------|
| `/o-m-cc:requirements <task>` | 要件定義 |
| `/o-m-cc:design` | 設計書作成 |
| `/o-m-cc:tasks` | タスク分解 |
| `/o-m-cc:plan <task>` | 上記を一括実行（scout によるギャップ分析含む） |

AskQuestionでサジェストが出るようになっているので、次のコマンドを選びやすい。

### 実行

| コマンド | 説明 |
|---------|------|
| `/o-m-cc:ultrawork <task>` | 並列エージェントで最大パフォーマンス実行 |
| `/o-m-cc:ultrawork-compact <task>` | /compact 後に ultrawork |
| `/o-m-cc:ultrawork-clear <task>` | /clear 後に ultrawork |

hooksではなくスラッシュコマンドでやっている。実行するとこんな感じ：

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

```bash
# コードレビュー（security-guidance連携 + code-simplifier提案）
/o-m-cc:review
```

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

12個のエージェントを用途別に使い分け：

### 計画系
| Agent | 役割 |
|-------|------|
| @analyst | 現状分析・要件定義 |
| @scout | ギャップ分析（必ず質問で終わる） |
| @designer | アーキテクチャ設計 |
| @planner | タスク分解 |
| @critic | 計画レビュー |

### 分析系
| Agent | 役割 |
|-------|------|
| @advisor | デバッグ・戦略相談 |
| @researcher | ドキュメント調査 |
| @explore | 高速コード探索 |
| @vision | PDF/画像分析 |

### 実装・品質系
| Agent | 役割 |
|-------|------|
| @frontend | UI/UXコンポーネント作成 |
| @document-writer | ドキュメント作成 |
| @code-reviewer | コードレビュー |

## 依存プラグイン

`/o-m-cc:sisyphus` 実行時に以下のプラグインを確認・インストール：

| プラグイン | 用途 |
|-----------|------|
| ralph-wiggum | `<promise>DONE</promise>` までループ継続 |
| frontend-design | フロントエンド設計支援 |
| code-simplifier | コード簡素化（レビュー後提案） |
| security-guidance | セキュリティレビュー支援 |

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

# 3. Sisyphus モードを有効化
/o-m-cc:sisyphus
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
- ultraworkで並列エージェント実行

GitHub: [kok1eee/o-m-cc](https://github.com/kok1eee/o-m-cc)

## 参考

- [cc-sdd](https://github.com/gotalab/cc-sdd) - 仕様駆動開発
- [oh-my-opencode](https://github.com/code-yeongyu/oh-my-opencode) - インスパイア元
- [ralph-wiggum](https://ghuntley.com/ralph/) - Sisyphus哲学
