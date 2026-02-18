---
title: "TAKT に学んだ3つのアイデアを Claude Code プラグインに取り入れた"
emoji: "🎵"
type: "tech"
topics: ["claudecode", "ai", "multiagent", "plugin", "takt"]
published: false
---

## はじめに

> 業務自動化Pythonエンジニア。バイブコーディング歴1年 ≒ エンジニア歴。

社内で AI オーケストレーションツール [TAKT](https://github.com/nrslib/takt) が話題になった。成瀬さんの記事を読んで設計思想を調べていたら、自分が作っている Claude Code プラグイン [o-m-cc](https://github.com/kok1eee/o-m-cc) に取り入れたいアイデアが3つ見つかった。

https://zenn.dev/nrs/articles/c6842288a526d7

実際に v0.15.0 として導入したので、TAKT の何に感銘を受けて、どう取り入れたかを書く。

前回の記事:

https://zenn.dev/kok1eee/articles/o-m-cc-subtraction-updates

## TAKT とは

成瀬さんが開発した AI エージェントオーケストレーションツール。「AI が言うことを聞いてくれない」「何度も指示しなければならない」という問題を、**構造で強制する**アプローチで解決している。

音楽のメタファーで設計されている:

- **Piece** — ワークフロー全体の定義（楽曲）
- **Movement** — ワークフロー内の1ステップ（楽章）
- **Persona** — エージェントの役割定義

YAML でワークフローを定義し、TypeScript エンジンが解釈・実行する。Claude Code、Codex、OpenCode など複数のAIプロバイダに対応。

## TAKT と o-m-cc の違い

同じ課題（AI に確実にタスクを遂行させる）を、異なるアプローチで解いている。

| | TAKT | o-m-cc |
|---|------|--------|
| 位置づけ | AI ツールの**外側**で制御 | Claude Code の**内側**で制御 |
| 定義方法 | YAML（厳密な構造） | Markdown（自然言語ベース） |
| 対応ツール | Claude Code / Codex / OpenCode | Claude Code 専用 |
| 導入 | TypeScript ランタイム | `claude plugin install` |
| ループ制御 | エンジンの `max_iterations` | Stop hook + `<promise>DONE</promise>` |
| 思想 | **構造で強制** | **peer-to-peer で自律** |

TAKT は汎用的で強力。o-m-cc は Claude Code に特化している分、追加のランタイムなしで動く。どちらが良いかではなく、チームのツールチェーン次第。

## 取り入れた3つのアイデア

TAKT の設計を調べる中で、o-m-cc にも活かせるアイデアが3つあった。

### 1. Faceted Prompting — 共通ポリシーの一元管理

#### TAKT の設計

TAKT はプロンプトを5つの独立した関心事に分離する:

- **Persona** — 誰として振る舞うか
- **Policy** — 何を守るか
- **Instruction** — 何をするか
- **Knowledge** — 何を参照するか
- **Output Contract** — どう出力するか

Policy を1つ変えれば、それを参照する全ワークフローに反映される。ソフトウェア設計の関心の分離と同じ発想。

#### o-m-cc での導入

o-m-cc には `code-reviewer` と `security-reviewer` という2つのレビューエージェントがある。両方に同じ Confidence Scoring の基準を書いていた:

```
# code-reviewer.md に書いてあった
| スコア | 確信度 | 報告 |
| 90-100 | 確実な問題 | 必ず報告 |
| 80-89  | 高い確信  | 報告 |
| 60-79  | 中程度    | 報告しない |

# security-reviewer.md にも同じ内容が書いてあった
```

これを `facets/policies/confidence-scoring.md` として抽出し、各エージェントから参照する形に変えた:

```markdown
# code-reviewer.md（変更後）
## Confidence Scoring
> **共通ポリシー**: `facets/policies/confidence-scoring.md` を Read して適用してください。
>
> Confidence 80以上の問題のみを報告。90+ = Critical、80-89 = Warning。
```

TAKT の Faceted Prompting をそのまま再現したわけではなく、**共通ポリシーの抽出・参照パターン**のエッセンスを取り入れた。TAKT なら YAML の `policies:` フィールドでエンジンが自動注入するが、o-m-cc ではエージェントに「Read して」と指示する形になる。Claude Code ネイティブなので、ファイルを読む行為自体はエージェントが普通にできる。

### 2. 集約ロジック — all()/any() による明示的な判定条件

#### TAKT の設計

TAKT では並列ステップの結果を集約する条件を明示的に定義できる:

```yaml
# TAKT のルール定義
rules:
  - condition: all("approved")
    next: complete
  - condition: any("needs_fix")
    next: fix
```

`all("approved")` は全員が承認したら次へ。`any("needs_fix")` は1人でも修正要求があったら差し戻し。曖昧さがない。

#### o-m-cc での導入

o-m-cc の `/review` スキルでは、2つのレビュアーの結果を集約する際に自然言語で書いていた:

```markdown
# Before（曖昧）
## 総合判定
→ 両方 Critical なし: マージ可能
→ いずれか Critical あり: 修正必須
```

これを TAKT 風の集約ルールに書き換えた:

```markdown
# After（明示的）
## 集約ルール
- all("Critical なし") → マージ可能
- any("Critical あり") → 修正必須。Step 5 へ
```

`/plan` スキルにも同様に導入:

```markdown
# Phase 1（Discovery Council）
Phase 1 集約ルール:
- all("報告完了") → analyst が requirements.md を最終確定

# Phase 4（Review Council）
Phase 4 集約ルール:
- all("承認") → 計画確定
- any("要修正") → 該当フェーズに差し戻し
```

TAKT では YAML エンジンがこのルールを機械的に評価するが、o-m-cc ではオーケストレーターの Lead がこの記述を読んで判断する。強制力は TAKT の方が高いが、**判定基準を明示する**だけでも再現性は上がる。

### 3. spawn prompt 構造統一 — コンテキスト/入力/出力の標準化

#### TAKT の設計

TAKT の movement は構造化されたプロンプトで動く。Persona、Policy、Instruction、Knowledge がそれぞれ独立したファイルから組み立てられ、毎回同じ構造で注入される。

#### o-m-cc での導入

o-m-cc の teammate spawn prompt はバラバラだった:

```markdown
# Before（バラバラ）
# code-reviewer の spawn prompt
agents/code-reviewer.md の指示に従い、以下の変更差分をレビューしてください。
## レビュー対象
[変更差分]
## チェック項目
- バグ、ロジックエラー
- ...
```

```markdown
# learnings-researcher の spawn prompt
agents/learnings-researcher.md の指示に従ってください。
## タスク
以下の機能に関連する過去の知見を検索してください
```

書く人（=自分）の気分で構造が変わる。これを統一した:

```markdown
# After（統一構造）
## エージェント定義
agents/code-reviewer.md の指示に従ってください。

## 参照ポリシー
facets/policies/confidence-scoring.md を Read して適用してください。

## コンテキスト
- タスク: $ARGUMENTS のコードレビュー
- スコープ: コード品質（バグ、複雑性、保守性）

## 入力
[変更差分を含める]

## チーム連携
- 発見した問題を security-reviewer にもメッセージで共有
- 完了したら Lead に結果サマリーをメッセージ送信

## 出力
- Confidence 80+ の問題のみ Critical/Warning で報告
```

6つのセクション: **エージェント定義 / 参照ポリシー / コンテキスト / 入力 / チーム連携 / 出力**。全9つの spawn prompt（review: 2、plan: 7）をこの構造に統一した。

## 導入してみて

### 良かった点

- **Confidence Scoring の変更が1箇所で済む** — facet を更新すれば両方のレビュアーに反映
- **集約ルールが読みやすい** — `all()`/`any()` は一目で判定条件がわかる
- **spawn prompt のレビューがしやすい** — 構造が統一されているので、何が書いてあるか / 何が抜けているかが明確

### TAKT との差

TAKT は YAML エンジンで**機械的に強制**するが、o-m-cc は Markdown で**意図を伝える**に留まる。集約ルールに `all("Critical なし")` と書いても、実際にそれを評価するのは Claude の判断。

この差は意図的なトレードオフ。o-m-cc は Claude Code のプラグインとして動くので、追加のランタイムを持たない。その代わり Claude Code の Agent Teams（TeammateTool）、hooks、タスクシステムを直接使う。

| | TAKT | o-m-cc |
|---|------|--------|
| 集約ルール | エンジンが機械的に評価 | Lead が判断 |
| ポリシー注入 | YAML で自動注入 | エージェントが Read |
| prompt 構造 | エンジンが組み立て | 人間が統一構造で記述 |
| 強制力 | **高い**（構造で強制） | **中**（意図を伝える） |
| 導入コスト | TypeScript ランタイム | プラグイン install |

## まとめ

TAKT から学んで o-m-cc v0.15.0 に取り入れたもの:

1. **Faceted Prompting** → `facets/policies/` で共通ポリシーを一元管理
2. **集約ロジック** → `all()`/`any()` で判定条件を明示
3. **spawn prompt 統一** → 6セクションの標準構造

本質的には**同じ課題を解いている**。「AI に確実にタスクを遂行させるには、仕様だけでなく強制力が必要」という洞察は共通。TAKT は外部エンジンで強制し、o-m-cc は Claude Code ネイティブに制御する。

TAKT の設計思想を公開してくれた成瀬さんに感謝。

## 参考

- [TAKT - AIの見張り番をやめよう](https://zenn.dev/nrs/articles/c6842288a526d7) - TAKT の設計思想
- [GitHub - nrslib/takt](https://github.com/nrslib/takt) - TAKT リポジトリ
- [GitHub - kok1eee/o-m-cc](https://github.com/kok1eee/o-m-cc) - o-m-cc リポジトリ
