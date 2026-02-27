---
title: "Claude Code の記憶設計 — auto-memory と HANDOVER.md の役割分担"
emoji: "🧠"
type: "tech"
topics: ["claudecode", "ai", "plugin", "automation"]
published: false
---

## はじめに

> 業務自動化Pythonエンジニア。バイブコーディング歴1年 ≒ エンジニア歴。

Claude Code コミュニティで HANDOVER.md（セッション引き継ぎ書）が少し流行った。セッション終了時に「今やっていたこと」を書き残して、次のセッションで読み込む。シンプルだが効果的なアイデアで、自分もいいと思って自作の Claude Code プラグイン [o-m-cc](https://github.com/kok1eee/o-m-cc) に取り入れた。

そこから発展させて HANDOVER.md の VCS 管理、履歴マイニング、自動スキル昇格と独自のナレッジ管理を作り込んだが、v0.17 でそれを全部捨てて auto-memory に移行し、HANDOVER.md を軽量版として復活させた。

「一回全部捨てて、必要なものだけ戻す」という流れだった。

前回の記事:

https://zenn.dev/kok1eee/articles/o-m-cc-claude-code-2149

## 背景：旧版の問題

v0.14 までの o-m-cc には、独自のナレッジ管理機構があった。

```
HANDOVER.md（VCS 管理）
  → セッション終了時に自動生成
  → jj で履歴管理
  → learnings-researcher が VCS 履歴をマイニング
  → promote-checker が繰り返しパターンを検出
  → スキルに自動昇格
```

一見よくできているが、3つの問題があった。

### 1. VCS 管理が重い

HANDOVER.md を VCS で管理していた理由は、`learnings-researcher` エージェントが過去の履歴を掘り返して知見を抽出するため。しかし実際には：

- 履歴が増えるほど検索が遅くなる
- 抽出された「知見」の質が安定しない
- コミットログが HANDOVER.md の更新で埋まる

### 2. 自動スキル昇格が過剰

v0.13 で導入した自動スキル昇格は「繰り返しパターンを自動検出→スキル化」という仕組みだった。しかし「このワークフローをスキルとして固定すべきか」は人間が判断すべき問題。自動化すると、不要なスキルが増える。

### 3. Claude Code 自身がやってくれる

Claude Code に `auto-memory` が実装された。セッション中に学んだパターンや慣習を `.claude/projects/*/memory/MEMORY.md` に自動で蓄積してくれる。つまり、**o-m-cc が独自に作っていたものと同じことを Claude Code 本体がやるようになった**。

## やったこと：全部捨てた

v0.17 で以下を廃止した。

| 廃止したもの | 理由 |
|-------------|------|
| HANDOVER.md の VCS 管理 | auto-memory が知識蓄積を担う |
| learnings-researcher | VCS 履歴マイニング不要 |
| promote-checker | 自動スキル昇格は過剰 |
| skill-candidates.md | 同上 |

廃止後の構成はシンプルになった。

```
auto-memory (MEMORY.md)  — 長期知識（パターン、慣習）
CLAUDE.md                — プロジェクトルール
```

## 廃止して気づいた2つのギャップ

全部捨てて動かしてみると、2つのギャップが見つかった。

### ギャップ1: compaction による記憶喪失

Claude Code はコンテキストウィンドウが上限に近づくと `auto-compaction` を実行する。古いメッセージを要約して圧縮する仕組みだが、**作業中の文脈（目標、進捗、判断の経緯）が圧縮で失われる**。

```
セッション開始
  → 複雑なタスクに着手
  → 途中で compaction 発生
  → 「何をやっていたか」が曖昧になる
  → 同じ調査を繰り返す
```

auto-memory は長期知識を保存するが、「今このセッションで何をやっているか」というセッション状態は保存しない。

### ギャップ2: cross-session の引き継ぎ

新しいセッションで「前回の続き」をやるとき、auto-memory だけでは不十分。MEMORY.md には「このプロジェクトは React + TypeScript で書かれている」のような知識はあるが、「昨日、認証機能のリファクタリングを半分まで進めて、残りは XYZ」のようなセッション状態はない。

## 解決：HANDOVER.md を軽量版で復活

旧版の問題（VCS 管理、自動昇格）は排除して、**セッション状態の保存だけに特化した HANDOVER.md を復活**させた。

### 役割分担

| 仕組み | 目的 | 寿命 |
|--------|------|------|
| auto-memory (MEMORY.md) | 長期知識（パターン、慣習） | 永続 |
| HANDOVER.md | セッション状態（目標、進捗、次ステップ） | 使い捨て |
| CLAUDE.md | プロジェクトルール | 永続 |

3つは並列の仕組みで、昇華パイプラインではない。HANDOVER.md の知見が自動的に MEMORY.md に昇格する仕組みは**意図的に作っていない**。

### 2系統の生成方法

#### 1. PreCompact hook（自動）

compaction 直前に発火する `PreCompact` hook で、transcript から機械的に HANDOVER.md を抽出する。

```bash
# hooks/pre-compact-handover.sh（抜粋）

# transcript から構造化データを抽出
FIRST_USER_MSG=$(grep '"role":"user"' "$TRANSCRIPT_PATH" | head -1 | \
  jq -r '.message.content | ...' | head -c 500)

CHANGED_FILES=$(grep '"tool_use"' "$TRANSCRIPT_PATH" | \
  jq -r 'select(.message.content[]?.name == "Write" or ...) | ...' | \
  sort -u | head -20)

LAST_ASSISTANT=$(grep '"role":"assistant"' "$TRANSCRIPT_PATH" | tail -1 | \
  jq -r '.message.content | ...' | head -c 1000)
```

AI は使わない。shell + jq のみ。完璧な要約ではないが、compaction 後の文脈復元には十分。

#### 2. `/o-m-cc:handover` スキル（手動）

セッション終了前に手動で呼ぶ。Claude が文脈を理解した上でリッチな引き継ぎ書を生成する。

```
## 目標
## 完了した作業
## 未完了の作業
## 決定事項
## 捨てた選択肢と理由
## 既知の問題・注意点
## 次のステップ
## 変更ファイルマップ
```

PreCompact hook が「最低限の保険」、handover スキルが「丁寧な引き継ぎ」という関係。

### SessionStart hook で表示

セッション開始時に HANDOVER.md があればセクション見出しを表示する。

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📋 前セッションの引き継ぎ (HANDOVER.md)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ## 目標
  ## 完了した作業
  ## 次のステップ

詳細は HANDOVER.md を Read してください。
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

見出しだけ出して詳細は Read に委ねる。トークン消費を最小限にする Progressive Disclosure。

## 旧版との比較

| | 旧版 (v0.14) | 新版 (v0.17) |
|---|---|---|
| 知識蓄積 | HANDOVER.md VCS 履歴 + learnings-researcher | auto-memory（Claude Code 本体） |
| セッション状態 | HANDOVER.md（VCS 管理） | HANDOVER.md（使い捨て、.gitignore） |
| スキル昇格 | promote-checker で自動 | 人間が判断して手動 |
| compaction 対策 | なし | PreCompact hook |
| 構成要素 | 5つ（hook + agent + skill + VCS + candidates） | 3つ（hook × 2 + skill） |

## 自動スキル昇格をやめた理由

v0.13 で自動スキル昇格を入れて、v0.17 で外した。学んだこと：

**「パターンの検出」と「パターンの固定化」は別の判断。**

auto-memory は前者を担う。「このプロジェクトでは X というパターンが使われている」を記録してくれる。しかし「X をスキルとして固定すべきか」は人間が決めるべき。理由：

- パターンが頻出 ≠ スキル化に値する。一時的な頻出かもしれない
- スキル化すると固定される。プロジェクトの方針が変わったとき、スキルが邪魔になる
- 人間が「これは定型化したい」と思ったとき初めてスキルを作る方が、結果的にスキルの質が高い

## 設計判断のまとめ

今回の変更を通して固まった判断基準：

### Claude Code がやることは Claude Code に任せる

auto-memory は Claude Code 本体の機能。プラグインが同じことをやる理由がない。「Claude Code がまだやってくれないこと」だけをプラグインで補う。

### 自動化の線引き

| 自動化すべき | 人間が判断すべき |
|-------------|----------------|
| セッション状態の保存（PreCompact hook） | スキル化の判断 |
| 前セッションの引き継ぎ表示 | 何を長期知識にするか |
| パターンの記録（auto-memory） | パターンを固定化するか |

### 使い捨てを恐れない

HANDOVER.md を VCS 管理していたのは「情報を失いたくない」という心理。しかし本当に価値のある知識は auto-memory に自然に蓄積される。セッション状態は使い捨てでいい。

## まとめ

独自のナレッジ管理を全部捨てて auto-memory に移行し、足りなかった部分だけ HANDOVER.md で補った。

```
v0.14: 独自ナレッジ管理（VCS + agent + auto-promote）
  ↓ 全部捨てる
v0.16: auto-memory のみ
  ↓ ギャップ発見
v0.17: auto-memory + HANDOVER.md（軽量版）
```

前回の「引き算」の記事と同じ結論になる。**プラットフォームが提供する機能が充実したら、プラグインは引く。** プラグインの役割は「プラットフォームがまだカバーしていない隙間を埋める」こと。隙間が埋まったら素直に退く。

リポジトリ: https://github.com/kok1eee/o-m-cc
