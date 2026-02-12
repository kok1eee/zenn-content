---
title: "Agent Teams に Council パターンを導入して計画フェーズを強化する"
emoji: "🏛️"
type: "tech"
topics: ["claudecode", "ai", "multiagent", "plugin", "automation"]
published: true
---

## はじめに

> 業務自動化Pythonエンジニア。バイブコーディング歴1年 ≒ エンジニア歴。

自作プラグイン [o-m-cc](https://github.com/kok1eee/o-m-cc) を v0.10.0 にした。今回の目玉は **Council パターン**の導入。

前回の v0.8.0 で Agent Teams (TeammateTool) に全面移行した。エージェント同士が peer-to-peer でメッセージを送り合えるようになったが、実は `/plan` のフローは「並列だけど会話しない」状態だった。これを「同じ問題を多角的に同時分析し、peer-to-peer で議論する」Council パターンに変えた。

まだ主に自分用だが、マルチエージェントの設計パターンとして参考になれば。

https://zenn.dev/kok1eee/articles/claude-opus-46-agent-teams-teammatetool

https://zenn.dev/kok1eee/articles/o-m-cc-claude-code-plugin

## Council パターンとは

Anthropic のブログ「[Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)」で紹介されているマルチエージェントパターンの一つ。

### Pipeline（従来）

```
A → B → C → D
```

各エージェントが前工程の出力を待って順次実行。シンプルだが、前工程のミスが後工程に伝播する。

### Council

```
┌──────────────────────┐
│  A ◄─► B ◄─► C      │
│  同じ問題を多角的に   │
│  同時分析 + 議論      │
└──────────────────────┘
```

複数のエージェントが**同じ問題**を**異なる視点**で同時に分析し、peer-to-peer で findings を共有する。Lead が全員の意見を統合して結論を出す。

### 使い分け

| パターン | 向いている場面 |
|----------|--------------|
| **Pipeline** | 入力→出力が明確な変換作業 |
| **Council** | 多角的な視点が必要な分析・レビュー |

## 変更前: Pipeline のみ

v0.9.0 までの `/plan` フロー:

```
Phase 0.5 (learnings) ─┐
                        ├→ Phase 1.5 (scout) → Phase 2 (designer) → Phase 3 (planner)
Phase 1   (analyst)   ─┘
```

learnings-researcher と analyst は並列だったが、**peer-to-peer 通信なし**。scout は analyst の出力 (requirements.md) を待つ。

問題:
- learnings-researcher が過去の知見を見つけても、analyst に直接伝わらない
- scout は requirements.md が完成するまで何もできない
- 3者の知見が統合されるのは「たまたまファイルを読んだとき」

## 変更後: Council + Pipeline ハイブリッド

v0.10.0:

```
┌─────────────────────────────────────────────────────┐
│              Phase 1: Discovery Council               │
│                                                       │
│  learnings-researcher ◄─► analyst (Lead) ◄─► scout   │
│  peer-to-peer で findings を共有                      │
│  analyst が統合して requirements.md を確定             │
└─────────────────────────────────────────────────────┘
          │
          ▼ requirements.md
┌──────────────┐    ┌──────────────┐
│  Phase 2     │───▶│  Phase 3     │
│  設計        │    │  タスク分解  │
│  (designer)  │    │  (planner)   │
└──────────────┘    └──────────────┘
          │
          ▼
┌──────────────────────────────────────────┐
│         Phase 4: Review Council           │
│                                           │
│  critic (Lead) ◄─► advisor               │
│  peer-to-peer で指摘を共有               │
│  critic が統合してレビュー結果を確定      │
└──────────────────────────────────────────┘
```

入口（Discovery）と出口（Review）に Council、中間に Pipeline。多角的分析が最も効果的な箇所だけ Council にした。

## Discovery Council の仕組み

3エージェントを同時に spawn し、peer-to-peer で findings を共有する。

### 役割分担

| エージェント | 視点 | 役割 |
|-------------|------|------|
| **analyst (Lead)** | 何があるか | 要件の整理・統合・確定 |
| **scout** | 何が足りないか | ギャップ・漏れ・エッジケースの発見 |
| **learnings-researcher** | 過去に何を学んだか | 知見の提供 |

### 動作フロー

```
1. 3エージェントが同時に作業開始
2. learnings-researcher: 過去の知見を検索 → 見つけ次第 analyst と scout にメッセージ
3. scout: ユーザーの要求 + コードベースから直接ギャップ分析 → analyst にメッセージ
4. analyst: 自身の分析 + 両者の findings を統合 → requirements.md を確定
```

### 変更前との比較

**Before**: scout は requirements.md の完成を待つ
```
analyst が requirements.md を書く
    ↓ (待ち時間)
scout が requirements.md を読んでギャップ分析
```

**After**: scout は requirements.md を待たず、直接分析を開始
```
analyst: 要件を整理中...        ← 同時
scout: コードベースからギャップ発見中... ← 同時
learnings-researcher: 過去の知見検索中... ← 同時
    ↕ peer-to-peer で随時共有
analyst: 全員の findings を統合して requirements.md 確定
```

### spawn prompt の書き方

Council の鍵は spawn prompt。「誰に」「いつ」「何を」共有するかを明記する。

```markdown
# analyst (Lead) の spawn prompt
TeammateTool: spawnTeammate
  name: "analyst"
  prompt: |
    ## Council Lead（peer-to-peer）
    あなたは Discovery Council の Lead です。
    - 要件ドラフトの主要部分ができたら scout・learnings-researcher に
      メッセージで共有し、フィードバックを促してください
    - scout からのギャップ報告を受け取り、要件に反映してください
    - learnings-researcher からの過去知見を受け取り、要件に反映してください
    - 全員の findings を統合してから requirements.md を最終確定してください

    ## 確定前チェック
    requirements.md を Write する前に、scout と learnings-researcher からの
    報告を受信済みか確認してください。
```

ポイント:
- **Lead の明示**: 誰が最終成果物を書くかを決める
- **確定前チェック**: 全員の報告を待ってから確定するルール
- **双方向の指示**: Lead だけでなく、メンバーにも「Lead に共有」を明記

## Review Council の仕組み

計画の最終チェックも Council 化した。critic 単独より、advisor の戦略的視点を加えた方がレビューの質が上がる。

### 役割分担

| エージェント | 視点 | 役割 |
|-------------|------|------|
| **critic (Lead)** | 計画の品質 | 完全性・実現可能性・リスク・明確性 |
| **advisor** | 技術的妥当性 | アーキテクチャ懸念・代替案 |

```markdown
# critic (Lead) の spawn prompt
TeammateTool: spawnTeammate
  name: "critic"
  prompt: |
    ## Council Lead（peer-to-peer）
    あなたは Review Council の Lead です。
    - 主要な指摘を advisor にメッセージで共有し、
      戦略的観点からのフィードバックを促してください
    - advisor からのアーキテクチャ懸念・代替案を受け取り、
      レビューに反映してください
```

## どこに Council を使うべきか

全てを Council にする必要はない。使い分けの判断基準:

### Council が効く場面

- **多角的な視点が価値を生む** — 1人では見落とす問題がある
- **同じ入力を異なる専門性で分析** — 要件を「何があるか」「何が足りないか」「過去に何を学んだか」の3視点で同時分析
- **レビュー・検証** — 計画の品質 + 技術的妥当性を同時チェック

### Pipeline で十分な場面

- **入力→出力が明確** — requirements.md → design.md のような変換
- **1者の集中作業** — designer が設計書を書く作業に複数の視点は邪魔
- **依存関係が強い** — 前工程の出力がないと次工程が始められない

### o-m-cc での判断結果

| フェーズ | パターン | 理由 |
|----------|----------|------|
| Phase 1 Discovery | **Council** | 3つの異なる視点で同時分析 |
| Phase 2 Design | Pipeline | 1者の集中作業 |
| Phase 3 Tasks | Pipeline | design.md という明確な入力 |
| Phase 4 Review | **Council** | 2つの異なる視点で同時検証 |
| /review (実装後) | Council | code + security の同時レビュー |

## おまけ: Sisyphus Default Agent

v0.10.0 ではもう一つ大きな変更がある。**デフォルトエージェント**の導入。

Claude Code の作者の一人、Boris Cherny さんのツイートでこの機能を知った。

https://x.com/bcherny/status/2021700144039903699

Claude Code には `settings.json` の `"agent"` フィールドでデフォルトエージェントを設定する機能がある。メイン会話がそのエージェントの指示で動く。

### 何が嬉しいか

今まで Sisyphus の精神（タスク完了まで止まらない、検証してから完了する）は CLAUDE.md に書いていた。しかし:

| 置き場所 | 本来の役割 |
|----------|-----------|
| **CLAUDE.md** | プロジェクト固有の知識（技術スタック、規約） |
| **Default Agent** | エージェントの振る舞い（完遂マインド、判断基準） |

CLAUDE.md に行動指針を混ぜると、プロジェクトが変わるたびに再設定が必要。Default Agent なら**どのプロジェクトでも同じ振る舞い**が効く。

### sisyphus.md

```yaml
---
name: sisyphus
description: タスク完了まで止まらない Sisyphus エージェント
model: sonnet
---
```

中身は:
- **Plan or Act 判断**: タスクの複雑さで自動的に計画モードか直接実行か決める
- **専門エージェントへの委譲**: `/plan`, `/review`, `/ultrawork` など適切なコマンドを選ぶ
- **検証してから完了**: 「たぶん動くはず」は証拠ではない
- **`<promise>DONE</promise>`**: 全タスク完了・検証済みの時だけ

### 有効化

```bash
# プロジェクト初期化時に自動配置
/o-m-cc:init

# 手動で有効化
claude --agent sisyphus
```

または `.claude/settings.json`:

```json
{
  "agent": "sisyphus"
}
```

## おまけ2: 推奨パーミッション自動設定

これも Boris Cherny さんのツイートから。自分は既にほぼ全てのコマンドを許可しているが、o-m-cc をインストールした人がすぐに使えるように、`/init` で推奨パーミッションを自動設定するようにした。

https://x.com/bcherny/status/2021700144039903699

Sisyphus Loop で自動実行中に権限承認で止まるのがボトルネックだった。spec/ 配下の読み書きと、コードベース探索を事前承認することで、計画フェーズが権限プロンプトなしで回る。

```json
{
  "permissions": {
    "allow": [
      "Read(spec/**)",
      "Write(spec/**)",
      "Edit(spec/**)",
      "Read(agents/**)",
      "Read(commands/**)",
      "Read(templates/**)",
      "Glob(**)",
      "Grep(**)"
    ]
  }
}
```

## まとめ

v0.10.0 の変更:

1. **Council + Pipeline ハイブリッド** — 入口と出口に Council、中間に Pipeline
2. **Discovery Council** — 3エージェントが同時分析 + peer-to-peer 共有
3. **Review Council** — 2エージェントが同時レビュー + peer-to-peer 共有
4. **Sisyphus Default Agent** — 振る舞いを CLAUDE.md から分離
5. **推奨パーミッション** — Sisyphus Loop のボトルネック解消

Council パターンのポイント:
- **spawn prompt に peer-to-peer 共有の指示を明記する** — これがないと Agent Teams の意味がない
- **Lead を決める** — 最終成果物を書く人が Lead
- **確定前チェック** — 全員の報告を待ってから確定
- **全てを Council にしない** — Pipeline で十分な場面を見極める

GitHub: [kok1eee/o-m-cc](https://github.com/kok1eee/o-m-cc)

## 参考

- [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents) - Anthropic のエージェント設計パターン
- [Orchestrate teams of Claude Code sessions](https://code.claude.com/docs/en/agent-teams) - Agent Teams 公式ドキュメント
- [Claude Code tips from Boris Cherry](https://x.com/AiDeveloperTips) - Default Agent, パーミッション事前承認の着想元
