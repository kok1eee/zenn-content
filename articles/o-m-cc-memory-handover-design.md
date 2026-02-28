---
title: "HANDOVER.md をうまく使えていなかったので Entire.io を参考に3層で作り直した"
emoji: "🧠"
type: "tech"
topics: ["claudecode", "ai", "plugin", "automation"]
published: true
---

## はじめに

> 業務自動化Pythonエンジニア。バイブコーディング歴1年 ≒ エンジニア歴。

Claude Code には auto-memory がある。プロジェクトの慣習やワークアラウンドを `MEMORY.md` に蓄積して、次のセッションで活用する仕組みだ。これは便利だが、**解決しない問題がある**。

「今何を作業していたか」だ。

auto-memory が覚えるのは「このプロジェクトでは jj を使う」「テストは vitest」といった**長期的な知識**。一方で compaction が起きたとき、あるいは次のセッションに切り替えたとき、「さっきどのファイルを変更して、どこでハマって、次に何をやるつもりだったか」は消える。これは auto-memory の限界ではなく、**そもそも auto-memory が解決すべき課題ではない**。

- **auto-memory** = 知識の蓄積（パターン、慣習、ワークアラウンド）
- **session context** = 作業状態の保存（Intent、Outcomes、Friction）

この2つは別レイヤーの課題なのに、コミュニティで流行った HANDOVER.md はこれを1ファイルに混在させていた。自分も自作の Claude Code プラグイン [o-m-cc](https://github.com/kok1eee/o-m-cc) に取り入れたが、うまく使いこなせていなかった。

以前から気になっていた [Entire.io](https://entire.io) と [Agent Trace](https://github.com/cursor/agent-trace) の設計を改めて読み込んでみたら、「知識」と「文脈」を明確に分離する方向が見えた。

前回の記事:

https://zenn.dev/kok1eee/articles/o-m-cc-claude-code-2149

## HANDOVER.md の何がダメだったか

### 1. 「知識」と「文脈」を区別していなかった

HANDOVER.md に書かれる内容は2種類あった：

- 「vitest のカバレッジ設定はこうする」 → **知識**（auto-memory に入るべき）
- 「認証機能の実装中、JWT の検証でハマっている」 → **文脈**（セッション固有の状態）

これを1ファイルに混在させたから、どちらとしても中途半端になった。知識は auto-memory に任せるべきだし、文脈は別の仕組みで管理すべきだった。

### 2. 構造が決まっていない

「セッションの引き継ぎ書を書け」だけでは、Claude が何を書くかはセッションの内容次第。ある時は詳細すぎ、ある時は肝心なことが抜ける。

### 3. 1ファイルに詰め込みすぎ

当初は1ファイルに Snapshot を蓄積していた。5件溜まったらダイジェストに統合するロジックも入れた。だが、このファイルは SessionStart で毎回フルロードされる。肥大化するほどトークンを食う。

```
HANDOVER.md（蓄積型）
  → Snapshot 1（知識と文脈が混在）
  → Snapshot 2
  → ...
  → Snapshot 5 → ダイジェストに統合
  → 際限なく膨らむ
```

### 4. VCS 管理が裏目に出た

履歴を掘り返して知見を抽出する `learnings-researcher` エージェントを作ったが、実際には：

- 履歴が増えるほど検索が遅くなる
- 抽出された「知見」の質が安定しない
- コミットログが HANDOVER.md の更新で埋まる

VCS 管理は「情報を失いたくない」という心理から来ていたが、**本当に価値のある知識は auto-memory に自然に蓄積される**。session context の履歴管理を自前で持つ必要はなかった。

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

### 「知識」と「文脈」を分ける

auto-memory が解決するのは「知識の蓄積」。session context が解決するのは「作業状態の保存」。HANDOVER.md の失敗は、この2つを混在させたこと。**それぞれに適した仕組みを用意し、Learnings 軸で橋渡しする。**

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

Claude Code の「記憶」には2つのレイヤーがある。

| レイヤー | 仕組み | 解決する課題 |
|---|---|---|
| **知識** | auto-memory (MEMORY.md) | パターン・慣習・ワークアラウンドの蓄積 |
| **文脈** | session context (context.md) | 「今何をしていたか」の保存 |

HANDOVER.md がうまくいかなかったのは、この2つを混在させていたから。auto-memory は auto-memory に任せ、session context だけを Entire.io inspired の4軸 + 3層アーキテクチャで管理する。Learnings 軸が2つのレイヤーを橋渡しする。

```
知識（auto-memory）          文脈（session context）
  MEMORY.md                   context.md / chronicle.md / archive
  パターン、慣習               Intent, Outcomes, Friction
  長期的                      セッション固有
       ▲                        │
       └── Learnings 軸で橋渡し ─┘
```

リポジトリ: https://github.com/kok1eee/o-m-cc
