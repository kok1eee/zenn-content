---
title: "Claude Code の記憶設計 — HANDOVER.md が使えなかったので Entire.io を参考に作り直した"
emoji: "🧠"
type: "tech"
topics: ["claudecode", "ai", "plugin", "automation"]
published: true
---

## はじめに

> 業務自動化Pythonエンジニア。バイブコーディング歴1年 ≒ エンジニア歴。

Claude Code コミュニティで HANDOVER.md（セッション引き継ぎ書）が流行った。セッション終了時に「今やっていたこと」を書き残して、次のセッションで読み込む。シンプルだが効果的なアイデアで、自分も自作の Claude Code プラグイン [o-m-cc](https://github.com/kok1eee/o-m-cc) に取り入れた。

だが、うまく使えなかった。

情報が足りないか、多すぎるか。1ファイルに蓄積していくと肥大化する。かといって毎回上書きすると前の経緯が消える。VCS で履歴管理してみたが、コミットログが HANDOVER.md の更新で埋まる。削除しようかと思った。

ただ、以前から気になっていた [Entire.io](https://entire.io) と [Agent Trace](https://github.com/cursor/agent-trace) の設計を改めて読み込んでみたら、「捨てる」のではなく「作り直す」方向が見えた。

前回の記事:

https://zenn.dev/kok1eee/articles/o-m-cc-claude-code-2149

## HANDOVER.md の何がダメだったか

### 1. 何を書けばいいかわからない

「セッションの引き継ぎ書を書け」だけでは、Claude が何を書くかはセッションの内容次第。ある時は詳細すぎ、ある時は肝心なことが抜ける。**構造が決まっていない**のが問題だった。

### 2. 1ファイルに詰め込みすぎ

当初は1ファイルに Snapshot を蓄積していた。5件溜まったらダイジェストに統合するロジックも入れた。だが、このファイルは SessionStart で毎回フルロードされる。肥大化するほどトークンを食う。

```
HANDOVER.md（蓄積型）
  → Snapshot 1
  → Snapshot 2
  → ...
  → Snapshot 5 → ダイジェストに統合
  → Snapshot 6...
  → 際限なく膨らむ
```

### 3. VCS 管理が裏目に出た

履歴を掘り返して知見を抽出する `learnings-researcher` エージェントを作ったが、実際には：

- 履歴が増えるほど検索が遅くなる
- 抽出された「知見」の質が安定しない
- コミットログが HANDOVER.md の更新で埋まる

VCS 管理は「情報を失いたくない」という心理から来ていたが、**本当に価値のある知識は Claude Code の auto-memory に自然に蓄積される**。履歴管理の仕組みを自前で持つ必要はなかった。

## 参考にしたもの

Entire.io も Agent Trace も以前から知っていた。ただ「自分のプラグインにどう活かすか」を考えたのは、HANDOVER.md を削除しようとしたこのタイミングが初めてだった。

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

## 作り直し：3層アーキテクチャ

Entire.io の構造化サマリーと、「文脈を捨てない」という Agent Trace の思想を組み合わせて、HANDOVER.md を3層に再設計した。

### ファイル構成

```
.claude/
├── context.md          ← 最新1スナップショットのみ（常にロード）
├── chronicle.md        ← 直近30エントリの1行ダイジェスト
└── context-archive.md  ← 全量保管（読み込まない）
```

**context.md** は常に最新1つだけ。SessionStart で即座に読める。前のスナップショットは chronicle.md に1行に圧縮して退避される。chronicle.md が30件を超えたら、古いものが archive に落ちる。

### データフロー

```
compaction 発生（自動）
  → 既存 Snapshot を chronicle に1行圧縮して退避
  → chronicle が 30 超過なら archive に退避
  → context.md を新しい Snapshot で上書き

/o-m-cc:handover（手動）
  → 4軸で詳細な引き継ぎ書を生成
  → Learnings を MEMORY.md に反映
  → 繰り返しパターンがあれば Skill 提案

セッション開始
  → context.md の最新状態を表示
  → chronicle.md の直近5件を表示
```

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

## Learnings → MEMORY.md の橋渡し

旧版の記事では「HANDOVER.md の知見が自動的に MEMORY.md に昇格する仕組みは意図的に作っていない」と書いた。

作り直してみて考えが変わった。**Learnings 軸があると、長期的な知見が自然に抽出される。** それを MEMORY.md に反映しない手はない。

ただし旧版の `learnings-researcher`（VCS 履歴をマイニングして知見を抽出）とは違う。handover スキル内で Claude 自身が「この Learnings は長期的に価値があるか」を判断して MEMORY.md に追記する。VCS は使わない。シンプルに Read → 判断 → Write。

```
Learnings セクション
  → プロジェクト固有のパターン？ → MEMORY.md に追記
  → 一時的な情報？ → context.md に残すだけ
```

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

| | 旧版 (v0.14-v0.17) | 新版 |
|---|---|---|
| ファイル構造 | 1ファイル蓄積型 | 3層分離 |
| ロード時コスト | ファイルサイズに比例 | 常に最新1スナップショット分 |
| 構造 | 自由記述 | 4軸固定（Entire.io inspired） |
| 知識蓄積 | VCS 履歴マイニング or なし | Learnings → MEMORY.md |
| スキル昇格 | 自動 or なし | 提案のみ（人間承認） |
| 履歴 | VCS 依存 or 使い捨て | chronicle.md（30件） |

## 設計判断のまとめ

### 自由記述より固定軸

「セッションの引き継ぎ書を書け」は曖昧すぎる。Entire.io のサマリー構造を見て、**軸を固定すれば品質が安定する**とわかった。AI に何をさせるかは、プロンプトの構造で決まる。

### 3層で寿命を分ける

| 層 | 寿命 | 用途 |
|---|---|---|
| context.md | 最新1件 | 即座にロード |
| chronicle.md | 直近30件 | 最近の経緯を把握 |
| context-archive.md | 全量 | 必要なとき参照 |

全部を1ファイルに入れるから肥大化する。寿命で分ければ、**常にロードするファイルは常に軽い**。

### 「捨てる」と「作り直す」は違う

旧版の記事では「全部捨てて必要なものだけ戻す」と書いた。実際にやったのは「捨てる → 足りない → 参考文献を読んで設計し直す」だった。引き算も大事だが、引きすぎたら足す。ただし前と同じものを足すのではなく、別の設計で作り直す。

## まとめ

HANDOVER.md がうまく使えなかったのは、構造が決まっていなかったから。Entire.io の構造化サマリーと Agent Trace の文脈保存の思想を参考に、3層アーキテクチャで作り直した。

```
旧: HANDOVER.md（1ファイル蓄積、自由記述）
  ↓ うまく使えない、削除しようとした
  ↓ Entire.io / Agent Trace を参考に設計し直す
新: context.md / chronicle.md / archive（3層、4軸固定）
  + Learnings → MEMORY.md
  + Skill 提案（人間承認）
```

リポジトリ: https://github.com/kok1eee/o-m-cc
