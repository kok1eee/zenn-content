---
title: "oh-my-opencodeが使えなくなったのでClaude Code版を自作した"
emoji: "🪨"
type: "tech"
topics: ["claudecode", "ai", "cli", "plugin"]
published: false
---

## はじめに

> 業務自動化Pythonエンジニア。バイブコーディング歴1年 ≒ エンジニア歴。

### きっかけ

AnthropicがOpenCodeをBANした。

oh-my-opencodeという便利なプラグインがあったが、Claude Codeでは使えない。じゃあ自分で作ろう。

## 作ったもの：o-m-cc

**Sisyphus Loop for Claude Code** - TODOが完了するまで止まらないマルチエージェントワークフロー

### Sisyphusとは

ギリシャ神話のシーシュポス。永遠に岩を山頂に押し上げ続ける男。

このプラグインは「タスク完了まで決して止まらない」というマインドセットをClaude Codeに注入する。

### 仕様駆動開発（SDD）

複雑なタスクは計画から：

```
要件定義 → ギャップ分析 → 設計 → タスク分解 → 実装
```

`/o-m-cc:plan` コマンドで一括実行できる。

## コマンド

### セットアップ

```bash
# Sisyphus モードを有効化
# （依存プラグイン確認 + CLAUDE.md設定 + hooks設定）
/o-m-cc:sisyphus
```

### 計画フェーズ

| コマンド | 説明 |
|---------|------|
| `/o-m-cc:requirements <task>` | 要件定義 |
| `/o-m-cc:design` | 設計書作成 |
| `/o-m-cc:tasks` | タスク分解 |
| `/o-m-cc:plan <task>` | 上記を一括実行（scout によるギャップ分析含む） |

### 実行

| コマンド | 説明 |
|---------|------|
| `/o-m-cc:ultrawork <task>` | 並列エージェントで最大パフォーマンス実行 |
| `/o-m-cc:ultrawork-compact <task>` | /compact 後に ultrawork |
| `/o-m-cc:ultrawork-clear <task>` | /clear 後に ultrawork |

### 品質

```bash
# コードレビュー（security-guidance連携 + code-simplifier提案）
/o-m-cc:review
```

## ワークフロー

### /o-m-cc:plan の流れ

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  Phase 1     │───▶│  Phase 1.5   │───▶│  Phase 2     │───▶│  Phase 3     │───▶│  Phase 4     │
│  要件定義    │    │  ギャップ    │    │  設計        │    │  タスク分解  │    │  レビュー    │
│  (analyst)   │    │  (scout)     │    │  (designer)  │    │  (planner)   │    │  (critic)    │
└──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
```

**Phase 1.5 (scout)** が特徴的：
- Prometheus式インタビュー
- **必ず質問で終わる**（パッシブ終了禁止）
- 要件の漏れ・曖昧さを発見
- Criticalな質問が解決するまで続行

## サブエージェント

12個のエージェントを用途別に使い分け：

### 計画系
| Agent | 役割 |
|-------|------|
| @analyst | 現状分析・要件定義 |
| @scout | ギャップ分析（Prometheus式インタビュー） |
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

## こだわったポイント

### しっかりプランを作ってから一気にコードを作る

cc-sddを参考に、いきなりコードを書かせない設計にした。

```
要件定義 → ギャップ分析 → 設計 → タスク分解 → 実装
```

計画フェーズをしっかり踏んでから実装に入るので、手戻りが少ない。

### AskQuestionで終わるようにした

一番こだわったのはここ。

@scout エージェントは**必ず質問で終わる**ように設計した。「何か質問ありますか？」で終わるのではなく、具体的な質問を投げかけて終わる。

これにより：
- 要件の漏れを発見できる
- 曖昧な部分を明確にできる
- 実装前に認識のズレを防げる

### /sisyphusコマンドでセットアップ

インストーラーではなくpluginとして配布したかったので、セットアップ用の `/o-m-cc:sisyphus` コマンドを用意した。

実行すると：
- CLAUDE.md に Sisyphus 原則を追加
- hooks 設定（`<promise>DONE</promise>` までループ）
- 依存プラグインの確認・インストール

意外と拡張性があって、OpenCodeにあるようなLSP機能（pyright-lsp、typescript-lsp）もインストールするようにしている。

### 実装後にコードレビュー → 修正ループ

OpenCodeのralph-loopは実装して終わり。

o-m-ccは違う。実装完了後に**コードレビューを挟む**：

```
実装完了 → @code-reviewer（並列でセキュリティチェック等） → 問題あり？ → 修正ループへ
```

security-guidance や code-simplifier と連携して、セキュリティチェックやコード簡素化を並列で実行。問題があれば再度修正ループに突入する。

## 使ってみた感想

### 正しくサブエージェントに振り分けてくれる

タスクの内容に応じて適切なエージェントを選んでくれる。設計は @designer、コード探索は @explore、レビューは @code-reviewer といった具合。

### タスク分解がしっかりしている

cc-sdd を参考にしているので、計画フェーズでタスクがしっかり分解される：

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

現状は汎用的なエージェントのみ。各分野の専門家エージェントがいない。

今後はskillなどで拡張していく予定：
- Python専門家（python-master skill連携）
- GAS専門家（gas-master skill連携）
- セキュリティ専門家（security-master skill連携）

プラグインの構造上、skillとの連携が自然にできるので、必要に応じて専門知識を追加していける。

## まとめ

- oh-my-opencodeが使えなくなったので自作
- Sisyphus哲学でTODO完了まで止まらない
- Prometheus式インタビューで要件の漏れを発見
- ultraworkで並列エージェント実行

GitHub: [kok1eee/o-m-cc](https://github.com/kok1eee/o-m-cc)

## 参考

- [cc-sdd](https://github.com/gotalab/cc-sdd) - 仕様駆動開発
- [oh-my-opencode](https://github.com/code-yeongyu/oh-my-opencode) - インスパイア元
- [ralph-wiggum](https://ghuntley.com/ralph/) - Sisyphus哲学
