---
title: "外部エンジン vs Claude Code ネイティブ — TAKT と o-m-cc のAIオーケストレーション比較"
emoji: "🎵"
type: "tech"
topics: ["claudecode", "ai", "multiagent", "plugin", "takt"]
published: true
---

## はじめに

> 業務自動化Pythonエンジニア。バイブコーディング歴1年 ≒ エンジニア歴。

社内で AI オーケストレーションツール [TAKT](https://github.com/nrslib/takt) が話題になった。「AI が言うことを聞かない問題」を解決するツールだという。

https://zenn.dev/nrs/articles/c6842288a526d7

調べてみたら面白かった。自分が作っている Claude Code プラグイン [o-m-cc](https://github.com/kok1eee/o-m-cc) と**同じ課題を全く違うアプローチで解いていた**。

- **TAKT**: AI ツールの**外側**に TypeScript エンジンを置いて制御する
- **o-m-cc**: Claude Code の**内側**（プラグイン + hooks）で制御する

この記事では、両者のアプローチの違いと、o-m-cc が仕様駆動開発（SDD）+ Sisyphus Loop でどう実現しているかを書く。

前回の記事:

https://zenn.dev/kok1eee/articles/o-m-cc-subtraction-updates

## 共通の課題: 「AI が言うことを聞かない」

AI エージェントに複雑なタスクを任せると、こういうことが起きる:

- 途中で脱線する
- 仕様と違うものを作る
- 「できました」と言うが実際は不完全
- 手戻りが多すぎて人間が監視し続ける必要がある

TAKT の記事のタイトルは「AIの見張り番をやめよう」。まさにこの問題を解決したい。

## TAKT のアプローチ: 外部エンジンで強制する

TAKT は AI ツールの外側に YAML エンジンを置く。

```
piece.yaml（ワークフロー定義）
    │
    ▼
TypeScript エンジン（ステートマシン）
    │
    ├── YAML → プロンプト組み立て
    ├── Claude Code / Codex を実行
    └── ルール評価 → 次のステップを決定
```

### YAML でワークフローを定義

音楽のメタファーで設計されている:

- **Piece** — ワークフロー全体（楽曲）
- **Movement** — 1ステップ（楽章）
- **Persona** — エージェントの役割

### 集約ルールで判定

並列ステップの結果を機械的に評価する:

```yaml
rules:
  - condition: all("approved")
    next: complete
  - condition: any("needs_fix")
    next: fix
```

全員が承認したら完了、1人でも修正要求があったら差し戻し。AI の判断ではなく、**エンジンが機械的に評価する**。

### Faceted Prompting

プロンプトを5つの関心事に分離する:

- **Persona** — 誰として振る舞うか
- **Policy** — 何を守るか
- **Instruction** — 何をするか
- **Knowledge** — 何を参照するか
- **Output Contract** — どう出力するか

Policy を変えれば全ワークフローに反映される。

### TAKT の強み

- **汎用的**: Claude Code、Codex、OpenCode に対応
- **強制力が高い**: YAML エンジンが機械的に制御するので AI が逸脱できない
- **再現性**: 同じ YAML なら同じフローが走る

## o-m-cc のアプローチ: Claude Code ネイティブに制御する

o-m-cc は Claude Code のプラグインシステムの中で動く。外部エンジンはない。

```
Claude Code
├── プラグイン (o-m-cc)
│   ├── agents/*.md      — エージェント定義
│   ├── skills/*/SKILL.md — スキル（ワークフロー）
│   └── facets/policies/  — 共通ポリシー
├── hooks               — イベント駆動の制御
│   ├── Stop hook        — 完了判定
│   └── SessionStart     — 状態復元
└── Agent Teams          — peer-to-peer マルチエージェント
```

追加のランタイムなし。Claude Code が持っている仕組みだけで動く。

### 仕様駆動開発（SDD）でレールを引く

TAKT が YAML でワークフローを定義するのに対し、o-m-cc は **仕様駆動開発でレールを引く**。

```
/plan「認証を実装して」
  → Phase 1: Discovery Council（要件定義）
     analyst + scout + learnings-researcher が同時分析
  → Phase 2: 設計（designer）
  → Phase 3: タスク分解（planner）
  → Phase 4: Review Council（レビュー）
     critic + advisor が同時検証
  → 出力: requirements.md, design.md, tasks.md
```

ここで作られた仕様（requirements, design, tasks）が AI の行動を制約する。仕様があるから「何を作るべきか」が明確で、脱線しにくい。

### Sisyphus Loop で完了まで止めない

仕様を引いた後は Sisyphus Loop が回る。

```
実装開始
  → TODO → 実装 → レビュー → 修正 → ...
  → Stop hook: <promise>DONE</promise> が出るまで止まらない
  → 最大イテレーション: SISYPHUS_MAX_ITERATIONS（デフォルト50）
```

TAKT が `max_iterations` + ルール評価で止めるのと同じことを、Claude Code の Stop hook で実現している。

### Agent Teams で並列レビュー

レビューは2つのエージェントが peer-to-peer で同時実行:

```
/review
  → code-reviewer + security-reviewer を同時 spawn
  → 互いの発見をメッセージで共有
  → 集約: all("Critical なし") → マージ可能
```

## 二つのアプローチの比較

| 観点 | TAKT | o-m-cc |
|------|------|--------|
| 位置 | AI ツールの**外側** | Claude Code の**内側** |
| 定義 | YAML | Markdown |
| 対応ツール | Claude Code / Codex / OpenCode | Claude Code 専用 |
| 導入 | TypeScript ランタイム | `claude plugin install` |
| レール | YAML ワークフロー | **仕様駆動開発**（SDD） |
| ループ | `max_iterations` | **Sisyphus Loop**（Stop hook） |
| 並列実行 | YAML の parallel substeps | **Agent Teams**（peer-to-peer） |
| 集約ルール | エンジンが機械的に評価 | Lead が判断 |
| 強制力 | **高い**（エンジンが強制） | **中**（仕様で制約 + hooks で制御） |

### どちらを選ぶか

- **複数の AI ツールを使い分ける** → TAKT（汎用的）
- **Claude Code に統一している** → o-m-cc（ネイティブで軽量）
- **厳密な制御が必要** → TAKT（機械的強制）
- **peer-to-peer の議論が欲しい** → o-m-cc（Agent Teams）

どちらが優れているという話ではなく、チームのツールチェーンと求める強制力のレベル次第。

## TAKT から学んで取り入れたもの

TAKT を調べる過程で、o-m-cc に取り入れたいアイデアも見つかった。v0.15.0 で以下の3つを導入した。

### 1. Faceted Prompting → 共通ポリシーの抽出

2つのレビューエージェントに同じ Confidence Scoring 基準を書いていたのを、`facets/policies/confidence-scoring.md` に抽出して参照する形に変更。

### 2. 集約ロジック → all()/any() の明示

レビューと計画の判定条件を `all("Critical なし")` / `any("要修正")` で明示化。

### 3. spawn prompt 統一 → 6セクション構造

全 teammate の spawn prompt を「エージェント定義 / 参照ポリシー / コンテキスト / 入力 / チーム連携 / 出力」に統一。

TAKT ではエンジンが機械的にこれらを実行するが、o-m-cc では Markdown で意図を伝える形。強制力は TAKT の方が高いが、**判定基準を明示するだけでも再現性は上がる**。

## まとめ

「AI が言うことを聞かない」問題に対するアプローチ:

```
TAKT:   YAML定義 + TypeScriptエンジン → 外から構造で強制
o-m-cc: SDD + Sisyphus Loop + Agent Teams → 中からネイティブに制御
```

TAKT は外部エンジンで強制力を担保する。o-m-cc は仕様駆動開発でレールを引き、Sisyphus Loop で完了まで走らせ、Agent Teams で並列レビューする。道具は違うが、解いている課題は同じ。

TAKT の設計思想を公開してくれた開発者の方に感謝。良い刺激になった。

## 参考

- [TAKT - AIの見張り番をやめよう](https://zenn.dev/nrs/articles/c6842288a526d7) - TAKT の設計思想
- [GitHub - nrslib/takt](https://github.com/nrslib/takt) - TAKT リポジトリ
- [GitHub - kok1eee/o-m-cc](https://github.com/kok1eee/o-m-cc) - o-m-cc リポジトリ
