---
title: "TAKT を調べて気づいた、Claude Code ネイティブでも同じ課題を解けるということ"
emoji: "🎵"
type: "tech"
topics: ["claudecode", "ai", "multiagent", "plugin", "takt"]
published: true
---

:::message
**TL;DR** — AI オーケストレーションツール TAKT と自作の o-m-cc を比較。「外部エンジン」と「Claude Code ネイティブ」という異なるアプローチで同じ課題に取り組んでいた。
:::

## はじめに

> 業務自動化Pythonエンジニア。バイブコーディング歴1年 ≒ エンジニア歴。

社内で AI オーケストレーションツール [TAKT](https://github.com/nrslib/takt) が話題になった。「AI が言うことを聞かない問題」を解決するツールだという。

https://zenn.dev/nrs/articles/c6842288a526d7

調べてみたら設計思想が非常に面白かった。そして気づいた。自分が Claude Code プラグインとして個人で作っている [o-m-cc](https://github.com/kok1eee/o-m-cc) が、アプローチは全く違うものの、同じ課題に取り組んでいたということに。

TAKT は実績のあるツールで、自分の o-m-cc はまだ主に自分用。比較するのはおこがましいが、「外部エンジン」と「Claude Code ネイティブ」という2つのアプローチの違いが面白かったので書いてみる。

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
- **実績がある**: 実際に使われている

## o-m-cc は Claude Code ネイティブで似たことをやっていた

TAKT を調べていて、自分の o-m-cc が別のアプローチで同じ課題に取り組んでいることに気づいた。

o-m-cc は Claude Code のプラグインシステムの中で動く。外部エンジンはない。まだ自分用で、TAKT のような実績はない。ただ、アプローチの違いが面白いので紹介する。

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

追加のランタイムなし。Claude Code が持っている仕組みだけで動く。Claude Code 専用という制約がある代わりに、導入は `claude plugin install` だけ。

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

TAKT の YAML ほどの強制力はない。仕様を渡しても AI が逸脱する可能性はある。ただ、レールがないよりはずっと良い。

### Sisyphus Loop で完了まで止めない

仕様を引いた後は Sisyphus Loop が回る。

```
実装開始
  → TODO → 実装 → レビュー → 修正 → ...
  → Stop hook: <promise>DONE</promise> が出るまで止まらない
  → 最大イテレーション: SISYPHUS_MAX_ITERATIONS（デフォルト50）
```

TAKT が `max_iterations` + ルール評価で完了判定するのと似た仕組みを、Claude Code の Stop hook で実現している。ただし TAKT はエンジンが機械的に判定するのに対し、o-m-cc は AI 自身が `<promise>DONE</promise>` を出すかどうかに依存する。強制力の差はここに出る。

### Agent Teams で並列レビュー

レビューは2つのエージェントが peer-to-peer で同時実行:

```
/review
  → code-reviewer + security-reviewer を同時 spawn
  → 互いの発見をメッセージで共有
  → 集約: all("Critical なし") → マージ可能
```

## アプローチの違い

| 観点 | TAKT | o-m-cc |
|------|------|--------|
| 位置 | AI ツールの**外側** | Claude Code の**内側** |
| 定義 | YAML（厳密） | Markdown（自然言語） |
| 対応ツール | Claude Code / Codex / OpenCode | Claude Code 専用 |
| 強制力 | **高い**（エンジンが機械的に強制） | **中**（仕様で制約 + hooks で制御） |
| 成熟度 | 実績あり | 個人利用レベル |

TAKT の方が汎用的で強制力が高い。o-m-cc は Claude Code に特化している分、導入が軽い。

## TAKT から学んで取り入れたもの

TAKT を調べる中で、o-m-cc に取り入れたいアイデアが見つかった。v0.15.0 で以下の3つを導入した。

### 1. Faceted Prompting → 共通ポリシーの抽出

2つのレビューエージェントに同じ Confidence Scoring 基準を書いていたのを、`facets/policies/confidence-scoring.md` に抽出して参照する形に変更。TAKT なら YAML エンジンが自動注入するが、o-m-cc ではエージェントに「Read して」と指示する形。

### 2. 集約ロジック → all()/any() の明示

レビューと計画の判定条件を `all("Critical なし")` / `any("要修正")` で明示化。TAKT ではエンジンが機械的に評価するが、o-m-cc ではオーケストレーターが読んで判断する。強制力は劣るが、判定基準を明示するだけでも再現性は上がる。

### 3. spawn prompt 統一 → 6セクション構造

全 teammate の spawn prompt を「エージェント定義 / 参照ポリシー / コンテキスト / 入力 / チーム連携 / 出力」に統一。TAKT の Faceted Prompting に触発された。

## まとめ

TAKT を調べて学んだこと:

- **「AI が言うことを聞かない」問題は構造で解決できる**。TAKT は外部エンジンで、o-m-cc は SDD + hooks で
- **強制力のレベルに差がある**。TAKT のエンジンによる機械的強制は強力。o-m-cc は仕様とhooksによる緩やかな制御
- **良いアイデアはアプローチが違っても取り入れられる**。Faceted Prompting、集約ロジック、prompt 構造統一は TAKT の設計から学んだ

o-m-cc はまだ個人プロジェクトだが、TAKT のような先行事例を参考にしながら改善を続けていきたい。

TAKT の設計思想を公開してくれた開発者の方に感謝。

## 参考

- [TAKT - AIの見張り番をやめよう](https://zenn.dev/nrs/articles/c6842288a526d7) - TAKT の設計思想
- [GitHub - nrslib/takt](https://github.com/nrslib/takt) - TAKT リポジトリ
- [GitHub - kok1eee/o-m-cc](https://github.com/kok1eee/o-m-cc) - o-m-cc リポジトリ
