---
title: "HANDOVER.md をうまく使えていなかったので Entire.io を参考に3層で作り直した"
emoji: "🧠"
type: "tech"
topics: ["claudecode", "ai", "plugin", "automation"]
published: true
---

:::message
**TL;DR** — HANDOVER.md を「知識の蓄積」と「セッション文脈の保存」に分離。Entire.io を参考に context.md + chronicle.md + MEMORY.md の3層アーキテクチャで作り直した。
:::

## はじめに

> 業務自動化Pythonエンジニア。バイブコーディング歴1年 ≒ エンジニア歴。

Claude Code コミュニティで HANDOVER.md（セッション引き継ぎ書）が流行った。自分も自作の Claude Code プラグイン [o-m-cc](https://github.com/kok1eee/o-m-cc) に取り入れたが、うまく使いこなせていなかった。

振り返ると、HANDOVER.md で解決したかった課題はそもそも2つあった。

| 課題 | 内容 | 例 |
|------|------|---|
| **知識の蓄積** | パターン、慣習、ワークアラウンドを覚える | 「このPJでは jj を使う」「テストは vitest」 |
| **セッション文脈の保存** | 今何をしていたか、どこでハマったかを残す | 「JWT 検証でハマっている」「次は○○をやる」 |

HANDOVER.md はこの2つを1ファイルに混在させていた。結果、どちらとしても中途半端だった。

整理してみたら、それぞれに適した解決策がすでにあった：

- **知識の蓄積** → Claude Code の auto-memory（MEMORY.md + skill + rules）で解決済み
- **セッション文脈の保存** → [Entire.io](https://entire.io) のアプローチを参考に3層アーキテクチャで解決

この記事は、後者の「セッション文脈の保存」をどう設計したかの話。

前回の記事:

https://zenn.dev/kok1eee/articles/o-m-cc-claude-code-2149

## HANDOVER.md で何がうまくいかなかったか

### 1. 構造が決まっていない

「セッションの引き継ぎ書を書け」だけでは、Claude が何を書くかはセッションの内容次第。ある時は詳細すぎ、ある時は肝心なことが抜ける。

### 2. 1ファイルに詰め込みすぎ

当初は1ファイルに Snapshot を蓄積していた。5件溜まったらダイジェストに統合するロジックも入れた。だが、このファイルは SessionStart で毎回フルロードされる。肥大化するほどトークンを食う。

```
HANDOVER.md（蓄積型）
  → Snapshot 1（知識と文脈が混在）
  → Snapshot 2
  → ...
  → Snapshot 5 → ダイジェストに統合
  → 際限なく膨らむ
```

### 3. VCS 管理が裏目に出た

履歴を掘り返して知見を抽出する `learnings-researcher` エージェントを作ったが、実際には：

- 履歴が増えるほど検索が遅くなる
- 抽出された「知見」の質が安定しない
- コミットログが HANDOVER.md の更新で埋まる

VCS 管理は「情報を失いたくない」という心理から来ていたが、**知識の蓄積は auto-memory に任せればいい**。session context の履歴管理を自前で持つ必要はなかった。

## 参考にしたもの

知識の蓄積は auto-memory で解決済み。残る課題は「セッション文脈の保存」。

Entire.io も Agent Trace も以前から知っていた。ただ「自分のプラグインにどう活かすか」を考えたのは、HANDOVER.md の課題を2つに分解できたこのタイミングが初めてだった。

### Entire.io — AI セッションの構造化

[Entire.io](https://entire.io) は元 GitHub CEO の Thomas Dohmke が作った AI-native version control ツール。Git を置き換えるのではなく、Git の上に「AI の思考過程」を記録するレイヤーを被せる。

Entire.io が面白いのは、Checkpoint（AI セッションの記録単位）ごとに **AI サマリーを自動生成**する点。そのサマリーが以下の軸で構造化されている。

- **Intent** — 何をしようとしていたか
- **Outcome** — 何が達成されたか
- **Learnings** — 何がわかったか
- **Friction** — 何にハマったか
- **Open Items** — 何が残っているか

改めて読み込んで「**セッションの記録に必要なのは、自由記述ではなく固定された軸だ**」と腹落ちした。

### Agent Trace — コードと文脈の紐付け

[Agent Trace](https://github.com/cursor/agent-trace) は AI 生成コードの帰属（attribution）を記録するオープン仕様。「このコードは誰が（人間 or AI）、どの会話で、どのモデルを使って書いたか」をファイル・行レベルで追跡する。

直接使うわけではないが、**コードだけ保存して文脈を捨てるのは、後から見たとき「なぜこうなったか」がわからない**という Provenance Gap の問題提起を、改めて自分の HANDOVER.md に当てはめてみた。解決すべきなのはまさにこれだった。

## 作り直し：session context の3層アーキテクチャ（v0.18）

auto-memory は auto-memory に任せる。session context だけを専用の仕組みで管理する。Entire.io の構造化サマリーと、「文脈を捨てない」という Agent Trace の思想を組み合わせて、o-m-cc v0.18 で3層に再設計した。

### ファイル構成

```
.claude/
├── context.md          ← 最新1スナップショットのみ（常にロード）
├── chronicle.md        ← 直近30エントリの1行ダイジェスト
└── context-archive.md  ← 全量保管（読み込まない）
```

**context.md** は常に最新1つだけ。SessionStart で即座に読める。前のスナップショットは chronicle.md に1行に圧縮して退避される。chronicle.md が30件を超えたら、古いものが archive に落ちる。

### hooks による自動化

3層のローテーションは hooks で自動化されている。人間もClaude も何もしなくていい。

| hook | イベント | やること |
|---|---|---|
| `pre-compact-handover.sh` | PreCompact | compaction 時に Snapshot を自動保存・ローテーション |
| `session-resume.sh` | SessionStart | context.md + chronicle.md の直近5件を表示 |

```
PreCompact hook（compaction が起きるたびに自動実行）
  1. context.md に既存 Snapshot があれば
     → 1行に圧縮して chronicle.md の先頭に追記
  2. chronicle.md が 30 超過
     → 超過分を context-archive.md に退避
  3. context.md を新しい Snapshot で上書き

SessionStart hook（セッション開始時に自動実行）
  1. context.md の Intent / Outcomes を表示
  2. chronicle.md の直近5件を表示（最近の経緯がわかる）
```

手動の `/o-m-cc:handover` スキルは、4軸で詳細な引き継ぎ書を生成する。加えて Learnings の MEMORY.md 反映と、繰り返しパターンの Skill 提案も行う。

### 4軸の構造

Entire.io の5軸から Open Items を Next Steps に読み替えて、Changed Files を追加。

| 軸 | 内容 |
|---|---|
| **Intent** | このセッションで何をしようとしていたか |
| **Outcomes** | 何が完了したか |
| **Learnings** | 確定した設計判断、発見したパターン |
| **Friction** | ハマったこと、失敗したアプローチ |
| **Next Steps** | 次のセッションで最初にやるべきこと |
| **Changed Files** | 変更ファイルと概要 |

Claude に「引き継ぎ書を書け」と言うのではなく、「この6つの軸で書け」と言う。**軸が固定されているから、毎回同じ品質の引き継ぎ書が出る。**

## Learnings → MEMORY.md：2つのレイヤーの橋渡し

「知識」と「文脈」は別レイヤーだが、完全に分断されているわけではない。session context の中に、長期的な知識として昇格すべき情報が含まれることがある。

例えば「JWT の検証で HS256 と RS256 を間違えてハマった」という Friction は、session context としてはそのセッション限りの話だが、「このプロジェクトでは RS256 を使う」という知識は auto-memory に入れるべきだ。

4軸の **Learnings** がこの橋渡しの役割を果たす。handover スキル内で Claude 自身が「この Learnings は長期的に価値があるか」を判断して MEMORY.md に追記する。

```
Session Context (context.md)          Knowledge (MEMORY.md)
  │                                     ▲
  │ Learnings 軸                        │
  └─── 長期的価値あり？ ──── YES ───────┘
                             NO → context.md に残すだけ
```

旧版の `learnings-researcher`（VCS 履歴をマイニングして知見を抽出）とは違い、VCS は使わない。シンプルに Read → 判断 → Write。

## Skill 提案の再導入

旧版では「自動スキル昇格」を入れて外した。「パターンの検出とパターンの固定化は別の判断」というのは今も正しいと思っている。

ただ、「提案する」のと「自動で作る」のは違う。

```
### Skill Suggestion
- 名前: sync-versions
- トリガー: バージョン番号を変更するとき
- 内容: plugin.json, marketplace.json, README.md のバージョンを同期
- 根拠: 過去3セッションで毎回同じ操作をしている
```

人間が承認したら skills/ に作成する。自動作成はしない。これなら「パターンの固定化は人間が判断すべき」という原則を守りつつ、気づきを逃さない。

## 旧版との比較

| | 旧版 (v0.14-v0.17) | v0.18 |
|---|---|---|
| 知識と文脈 | 混在（HANDOVER.md 1ファイル） | 分離（auto-memory / context.md） |
| ファイル構造 | 1ファイル蓄積型 | context.md / chronicle.md / archive の3層 |
| ロード時コスト | ファイルサイズに比例 | 常に最新1スナップショット分 |
| 構造 | 自由記述 | 4軸固定（Entire.io inspired） |
| 自動化 | なし（手動 or スキル） | PreCompact hook で自動ローテーション |
| 知識 → auto-memory | VCS 履歴マイニング or なし | Learnings 軸で橋渡し |
| スキル昇格 | 自動 or なし | 提案のみ（人間承認） |
| 履歴 | VCS 依存 or 使い捨て | chronicle.md（直近30件保持） |

## 設計判断のまとめ

### 課題を分解する

HANDOVER.md がうまくいかなかったのは、2つの課題を1つの仕組みで解決しようとしたこと。**知識は auto-memory に任せ、セッション文脈だけを専用の仕組みで管理する。** Learnings 軸が2つを橋渡しする。

### 自由記述より固定軸

「セッションの引き継ぎ書を書け」は曖昧すぎる。Entire.io のサマリー構造を見て、**軸を固定すれば品質が安定する**とわかった。AI に何をさせるかは、プロンプトの構造で決まる。

### 3層で寿命を分ける

| 層 | 寿命 | 用途 |
|---|---|---|
| context.md | 最新1件 | 即座にロード |
| chronicle.md | 直近30件 | 最近の経緯を把握 |
| context-archive.md | 全量 | 必要なとき参照 |

全部を1ファイルに入れるから肥大化する。寿命で分ければ、**常にロードするファイルは常に軽い**。

## まとめ

HANDOVER.md で解決したかった課題は2つあった。

| 課題 | 解決策 |
|------|--------|
| **知識の蓄積** | auto-memory（MEMORY.md + skill + rules）— すでにある |
| **セッション文脈の保存** | Entire.io inspired の4軸 + 3層アーキテクチャ — 今回作った |

HANDOVER.md がうまくいかなかったのは、この2つを1ファイルに混ぜていたから。課題を分解したら、知識は auto-memory で解決済みだった。残るセッション文脈だけを Entire.io のアプローチで設計し直した。

```
課題1: 知識の蓄積            課題2: セッション文脈の保存
  → auto-memory で解決済み     → Entire.io 参考に3層で解決
  MEMORY.md / skill / rules    context.md / chronicle.md / archive
       ▲                        │
       └── Learnings 軸で橋渡し ─┘
```

リポジトリ: https://github.com/kok1eee/o-m-cc
