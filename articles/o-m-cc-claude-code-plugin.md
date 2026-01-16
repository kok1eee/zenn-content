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
要件定義 → 設計 → タスク分解 → 実装
```

`/plan` コマンドで一括実行できる。

## コマンド

### セットアップ

```bash
# Sisyphus モードを有効化（CLAUDE.md に追記）
/sisyphus
```

### 計画フェーズ

| コマンド | 説明 |
|---------|------|
| `/requirements <task>` | 要件定義 |
| `/design` | 設計書作成 |
| `/tasks` | タスク分解 |
| `/plan <task>` | 上記を一括実行 |

### 品質

```bash
# コードレビュー
/review
```

## サブエージェント

11個のエージェントを用途別に使い分け：

### 計画系
| Agent | 役割 |
|-------|------|
| @analyst | 現状分析・要件定義 |
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

一度計画を承認したら、あとは「実装開始して」と言うだけ。TODO完了まで止まらずに進めてくれるので、他の作業をしながら待てる。

## 今後やりたいこと

- 「実装開始しますか？」でultraworkモードを選べるようにする
  - 通常モード：確認しながら進める
  - ultraworkモード：全部任せて一気に進める

## インストール

```bash
# プラグインをインストール
claude plugin install kok1eee/o-m-cc

# Sisyphus モードを有効化
/sisyphus
```

## まとめ

- oh-my-opencodeが使えなくなったので自作
- Sisyphus哲学でTODO完了まで止まらない
- 仕様駆動開発で複雑なタスクも構造化

GitHub: [kok1eee/o-m-cc](https://github.com/kok1eee/o-m-cc)

## 参考

- [cc-sdd](https://github.com/gotalab/cc-sdd) - 仕様駆動開発
- [oh-my-opencode](https://github.com/code-yeongyu/oh-my-opencode) - インスパイア元
- [ralph-wiggum](https://ghuntley.com/ralph/) - Sisyphus哲学
