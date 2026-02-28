---
title: "Claude Code の auto-memory とセッション引き継ぎを設計する"
emoji: "🧠"
type: "tech"
topics: ["claudecode", "ai", "automation", "tips"]
published: true
---

## はじめに

> 業務自動化Pythonエンジニア。バイブコーディング歴1年 ≒ エンジニア歴。

Claude Code の auto-memory について、公式ドキュメントには基本的な使い方が書かれているが、**設計思想や実践的なノウハウ**は少ない。自作プラグイン [o-m-cc](https://github.com/kok1eee/o-m-cc) の開発で独自のナレッジ管理を作り、全部捨てて auto-memory に移行し、そこから足りない部分を見つけて補完した経験をまとめる。

この記事は Claude Code ユーザー全般向け。プラグインを使っていなくても適用できる内容。

## Claude Code の記憶の全体像

Claude Code には複数の「記憶」がある。公式ドキュメントに記載されているものを整理する。

### 記憶の種類

| 種類 | 場所 | 書く人 | 読み込み | 共有 |
|------|------|--------|---------|------|
| Project memory | `./CLAUDE.md` | 人間 | 起動時に全文 | チーム（VCS） |
| Project rules | `./.claude/rules/*.md` | 人間 | 起動時に全文 | チーム（VCS） |
| User memory | `~/.claude/CLAUDE.md` | 人間 | 起動時に全文 | 自分のみ（全PJ） |
| Project local | `./CLAUDE.local.md` | 人間 | 起動時に全文 | 自分のみ（当PJ） |
| **Auto memory** | `~/.claude/projects/<project>/memory/` | **Claude** | 起動時に200行 | 自分のみ（当PJ） |

ポイントは、**auto-memory だけが Claude 自身が書く記憶**であること。他はすべて人間が書く。

### auto-memory の仕組み

auto-memory はセッション中に Claude が自律的に `.claude/projects/<project>/memory/MEMORY.md` に書き込む。人間が「覚えて」と言えば書くし、Claude が自分で「これは記録すべき」と判断しても書く。

```
~/.claude/projects/<project>/memory/
├── MEMORY.md          # インデックス（200行制限）
├── debugging.md       # トピックファイル（任意）
├── api-conventions.md # トピックファイル（任意）
└── ...
```

**MEMORY.md は200行制限がある。** 超えると最初の200行だけ読み込まれ、残りは切り捨てられる。詳細はトピックファイルに分割して、MEMORY.md はインデックスとして使うのが公式の推奨。

トピックファイルは起動時に読み込まれない。Claude が必要なときに Read で読む。

## 実践：auto-memory を使いこなす

### 1. 明示的に覚えさせる

一番確実な方法。

```
「このプロジェクトでは pnpm を使う。npm は使わない。覚えて」
「API テストにはローカルの Redis が必要。memory に保存して」
```

Claude は言われた内容を MEMORY.md に書き込む。明示的に指示すれば確実に保存される。

### 2. 200行制限の運用

MEMORY.md が肥大化すると、200行を超えた部分は読み込まれない。

**やるべきこと**:
- MEMORY.md はインデックスとして簡潔に保つ
- 詳細はトピックファイルに分割する
- 定期的に `/memory` で内容を確認して整理する

```markdown
# MEMORY.md（インデックスとして）

## ビルド・テスト
- ビルド: `npm run build`
- テスト: `npm run test`
- 詳細: [testing.md](testing.md)

## アーキテクチャ
- React + TypeScript + Tailwind
- 詳細: [architecture.md](architecture.md)
```

### 3. 自分で MEMORY.md を編集する

auto-memory は Claude が書くものだが、**人間が直接編集しても良い**。`/memory` コマンドでエディタが開く。

Claude が書いた内容が冗長だったり、不正確だったら直接修正する。Claude が書く + 人間が校正する、という運用が安定する。

### 4. 設定による制御

auto-memory を無効化したい場合:

```json
// ~/.claude/settings.json（全プロジェクト）
{ "autoMemoryEnabled": false }

// .claude/settings.json（特定プロジェクト）
{ "autoMemoryEnabled": false }
```

環境変数で強制制御もできる:

```bash
export CLAUDE_CODE_DISABLE_AUTO_MEMORY=1  # 強制 OFF
export CLAUDE_CODE_DISABLE_AUTO_MEMORY=0  # 強制 ON
```

CI 環境では OFF にしておくのが安全。

## 「覚えること」は2つある

Claude Code の記憶設計を考えるとき、管理すべきものは最初から2つある。

| | 知識 | 文脈 |
|---|---|---|
| 例 | 「このPJでは pnpm を使う」 | 「認証機能の実装中、JWT 検証でハマっている」 |
| 寿命 | 永続（プロジェクトが続く限り） | 短命（セッション～数日） |
| 蓄積先 | auto-memory (MEMORY.md) | session context ファイル |

**知識**は auto-memory がカバーしてくれる。では**文脈**は？

### compaction でセッション文脈が消える

Claude Code はコンテキストウィンドウが上限に近づくと **auto-compaction** を実行する。古いメッセージを圧縮して、新しいメッセージのための空間を作る。

```
セッション開始
  → 「認証機能をリファクタリングして」
  → 調査、設計判断、途中まで実装
  → ← ここで compaction 発生
  → 「あれ、何をやっていたんだっけ...」
```

auto-memory が覚えるのは「パターン」であって「今やっていること」ではない。セッション文脈は別の仕組みで保存する必要がある。

## セッション状態を保存する

auto-memory とは別に、セッション状態を保存する仕組みが必要。ここでは2つのアプローチを紹介する。

### 記憶を2層に分ける

| 層 | 内容 | 寿命 | 保存先 |
|----|------|------|--------|
| **知識** | パターン、慣習、設定 | 永続 | auto-memory (MEMORY.md) |
| **文脈** | 今の目標、進捗、次ステップ | セッション～数日 | session context ファイル |

この2つは**別レイヤー**。セッション状態が自動的に長期知識に昇格する仕組みは不要。長期知識にすべきかどうかは auto-memory の判断（あるいは人間の判断）に任せる。

### アプローチ1: HANDOVER.md（シンプル版）

最もシンプルな方法。PreCompact hook で compaction 前にセッション状態を保存する。

```json
// .claude/settings.json
{
  "hooks": {
    "PreCompact": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/save-session-state.sh",
            "timeout": 10000
          }
        ]
      }
    ]
  }
}
```

hook の入力には `transcript_path`（会話履歴のファイルパス）が含まれる。ここから shell + jq で情報を抽出する。

```bash
#!/bin/bash
# .claude/hooks/save-session-state.sh
set -euo pipefail

HOOK_INPUT=$(cat)
TRANSCRIPT_PATH=$(echo "$HOOK_INPUT" | jq -r '.transcript_path // empty')

if [[ -z "$TRANSCRIPT_PATH" ]] || [[ ! -f "$TRANSCRIPT_PATH" ]]; then
  exit 0
fi

# 最初のユーザーメッセージ（≒ 目標）
FIRST_USER_MSG=$(grep '"role":"user"' "$TRANSCRIPT_PATH" | head -1 | \
  jq -r '.message.content | if type == "array" then
    map(select(.type == "text")) | map(.text) | join(" ")
  else . end' 2>/dev/null | head -c 500 || echo "(抽出失敗)")

# 変更ファイル一覧
CHANGED_FILES=$(grep '"tool_use"' "$TRANSCRIPT_PATH" | \
  jq -r 'select(.message.content[]?.name == "Write" or
    .message.content[]?.name == "Edit") |
    .message.content[] | select(.type == "tool_use") |
    .input.file_path // empty' 2>/dev/null | \
  sort -u | head -20 || echo "")

# 最後のアシスタントメッセージ（≒ 現在の状態）
LAST_ASSISTANT=$(grep '"role":"assistant"' "$TRANSCRIPT_PATH" | tail -1 | \
  jq -r '.message.content | if type == "array" then
    map(select(.type == "text")) | map(.text) | join("\n")
  else . end' 2>/dev/null | head -c 1000 || echo "(抽出失敗)")

# HANDOVER.md に保存
cat > HANDOVER.md << EOF
# Session Handover

> auto-compaction 前に自動生成

## 目標
$FIRST_USER_MSG

## 変更ファイル
$(echo "$CHANGED_FILES" | while IFS= read -r f; do
  [[ -n "$f" ]] && echo "- \`$f\`"; done)

## 最後のコンテキスト
$LAST_ASSISTANT
EOF

echo "📝 Session state saved to HANDOVER.md"
exit 0
```

AI は使わない。shell + jq のみで完結する。完璧な要約ではないが、compaction 後に「何をやっていたか」を思い出すには十分。

SessionStart hook で表示すれば、次のセッションでも引き継ぎできる。

```bash
#!/bin/bash
# SessionStart hook: HANDOVER.md があれば表示
set -euo pipefail
HOOK_INPUT=$(cat)

if [[ ! -f "HANDOVER.md" ]]; then
  exit 0
fi

echo ""
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "📋 前セッションの引き継ぎあり"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo ""
grep '^## ' HANDOVER.md 2>/dev/null | while IFS= read -r line; do
  echo "  ${line}"
done
echo ""
echo "詳細は HANDOVER.md を Read してください。"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
exit 0
```

HANDOVER.md は `.gitignore` に入れて使い捨てにする。セッション状態は揮発性データなので VCS 管理する意味はない。

### アプローチ2: 3層アーキテクチャ（o-m-cc v0.18）

自作プラグイン o-m-cc では、HANDOVER.md の運用で「1ファイルに蓄積すると肥大化する」「毎回上書きすると経緯が消える」という問題にぶつかった。

解決策として、session context を3層に分離した。

```
.claude/
├── context.md          ← 最新1スナップショットのみ（常にロード）
├── chronicle.md        ← 直近30エントリの1行ダイジェスト
└── context-archive.md  ← 全量保管（読み込まない）
```

| 層 | 寿命 | 用途 |
|---|---|---|
| context.md | 最新1件 | 即座にロード |
| chronicle.md | 直近30件 | 最近の経緯を把握 |
| context-archive.md | 全量 | 必要なとき参照 |

PreCompact hook でローテーションを自動化。compaction が起きるたびに：
1. 既存 Snapshot を chronicle.md に1行圧縮して退避
2. chronicle.md が30件超過なら archive に退避
3. context.md を新しい Snapshot で上書き

**常にロードするファイルは常に軽い。** 経緯も30件分は残る。

詳細は別記事で書いた：

https://zenn.dev/kok1eee/articles/o-m-cc-memory-handover-design

## 設計判断のまとめ

### 「知識」と「文脈」を混ぜない

auto-memory が解決するのは「知識の蓄積」。session context が解決するのは「作業状態の保存」。この2つを1ファイルに混在させると、どちらとしても中途半端になる。

### 自動化の線引き

| やること | 自動/手動 |
|---------|---------|
| パターンの記録 | 自動（auto-memory に任せる） |
| セッション状態の保存 | 自動（PreCompact hook） |
| パターンの固定化（スキル化） | 手動（人間が判断） |
| ルール化（CLAUDE.md 追記） | 手動（人間が判断） |

### VCS 管理は不要

セッション状態は揮発性データ。`.gitignore` に入れて使い捨てにする。長期的に価値のある知識は auto-memory に任せればよい。

## まとめ

Claude Code の「記憶」には2つのレイヤーがある。

| レイヤー | 仕組み | 解決する課題 |
|---|---|---|
| **知識** | auto-memory (MEMORY.md) | パターン・慣習・ワークアラウンドの蓄積 |
| **文脈** | session context (HANDOVER.md / context.md) | 「今何をしていたか」の保存 |

auto-memory にセッション状態を入れるのは用途違い。session context に長期知識を蓄積するのも用途違い。**それぞれに適した仕組みを用意して、混ぜない**のが設計のポイント。

```
知識（auto-memory）          文脈（session context）
  MEMORY.md                   HANDOVER.md or context.md
  パターン、慣習               Intent, Outcomes, Friction
  永続                        セッション固有
```

独自の記憶管理を作り込むより、**Claude Code のネイティブ機能を最大限使って、足りない部分だけ hook で補完する**のが結局一番うまくいく。

公式ドキュメント: [Manage Claude's memory](https://code.claude.com/docs/en/memory) | [Hooks reference](https://code.claude.com/docs/en/hooks)

リポジトリ: https://github.com/kok1eee/o-m-cc

---

## 後書き：そもそも compaction を減らす

ここまで「compaction が起きたときにどう対処するか」を書いたが、**compaction の発生頻度を下げる**アプローチもある。

[rtk](https://github.com/rtk-ai/rtk) は CLI コマンドの出力を圧縮して、コンテキストに入るトークン量を 60〜90% 削減する CLI プロキシ。`git status` や `pytest` の出力から必要な情報だけを残して、残りを捨てる。

```
# 圧縮例
cargo test: 155行 → 3行（98%削減）
git status: 119文字 → 28文字（76%削減）
```

仕組みは PreToolUse hook で Bash コマンドを透過的に `rtk <コマンド>` に書き換える。Claude Code 側は書き換えを意識しない。

対応コマンドは git, pytest, ruff, cargo, vitest, eslint, tsc, docker など。トークン消費が減る → コンテキストウィンドウに余裕ができる → **compaction の発生が遅くなる**。

PreCompact hook で「compaction が起きたときの対策」を入れつつ、rtk で「そもそも compaction が起きにくくする」。両方やるのが理想的。
