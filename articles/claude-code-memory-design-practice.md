---
title: "Claude Code の記憶はどう設計すべきか — auto-memory の実践と限界、補完手法"
emoji: "🧠"
type: "tech"
topics: ["claudecode", "ai", "automation", "tips"]
published: false
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

## auto-memory の限界

auto-memory を実運用してわかった限界が2つある。

### 限界1: compaction で「今やっていること」が消える

Claude Code はコンテキストウィンドウが上限に近づくと **auto-compaction** を実行する。古いメッセージを圧縮して、新しいメッセージのための空間を作る。

問題は、**作業中のコンテキストが圧縮で失われる**こと。

```
セッション開始
  → 「認証機能をリファクタリングして」
  → 調査、設計判断、途中まで実装
  → ← ここで compaction 発生
  → 「あれ、何をやっていたんだっけ...」
  → 同じ調査をもう一度やる
```

auto-memory は**長期知識**（パターン、慣習、設定値）を記録する。「今このセッションで何をやっているか」というセッション状態は保存しない。長いセッションで compaction が起きると、作業の文脈が失われる。

### 限界2: セッションをまたぐ引き継ぎができない

新しいセッションで「前回の続き」をやりたいとき、auto-memory だけでは不十分。

MEMORY.md には「このプロジェクトは React + TypeScript」のような知識はあるが、「昨日、認証機能のリファクタリングを半分まで進めて、残りは XYZ」のようなセッション状態はない。

```
MEMORY.md に書かれていること:
  ✅ ビルドコマンドは npm run build
  ✅ テストは vitest を使う
  ✅ APIは /api/v2 プレフィックス

MEMORY.md に書かれていないこと:
  ❌ 昨日のセッションで何をやったか
  ❌ どこまで進んだか
  ❌ 次に何をすべきか
```

## 補完：セッション状態を保存する

auto-memory の限界を補完する方法として、**セッション状態を別のファイルに保存する**アプローチがある。

### 考え方：記憶を2層に分ける

| 層 | 内容 | 寿命 | 保存先 |
|----|------|------|--------|
| 長期知識 | パターン、慣習、設定 | 永続 | auto-memory (MEMORY.md) |
| セッション状態 | 今の目標、進捗、次ステップ | 使い捨て | HANDOVER.md 等 |

この2つは**並列であり、昇華パイプラインではない**。セッション状態が自動的に長期知識に昇格する仕組みは作らない。長期知識にすべきかどうかは auto-memory の判断（あるいは人間の判断）に任せる。

### 方法1: PreCompact hook で自動保存

Claude Code 2.x で追加された `PreCompact` hook を使う。compaction が発生する直前に自動で発火する。

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

hook の入力には `transcript_path`（会話履歴のファイルパス）が含まれる。ここから必要な情報を抽出できる。

```json
// PreCompact hook が受け取る入力
{
  "session_id": "abc123",
  "transcript_path": "/Users/.../.claude/projects/.../session.jsonl",
  "cwd": "/Users/.../my-project",
  "hook_event_name": "PreCompact",
  "trigger": "auto",
  "custom_instructions": ""
}
```

transcript は JSONL 形式で、各行が1つのメッセージ。ここから shell + jq で情報を抽出する。

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

### 方法2: 手動で引き継ぎを書く

セッション終了前に Claude に引き継ぎ書を書かせる。

```
「セッションの引き継ぎ書を HANDOVER.md に書いて」
```

Claude が文脈を理解した上で、目標・完了作業・未完了作業・次のステップを構造化して書いてくれる。PreCompact hook の機械的な抽出よりもリッチな内容になる。

### 方法3: SessionStart hook で表示

保存した引き継ぎ書を次のセッションで読み込む。

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

SessionStart hook の matcher は `startup`, `resume`, `clear`, `compact` に対応している。セッション開始時に HANDOVER.md のセクション見出しだけを表示し、詳細は Read に委ねる。トークン消費を最小限にしつつ、Claude に「前回の続きがある」と伝える。

## HANDOVER.md は VCS で管理すべきか

結論：**管理しない方がいい**。

`.gitignore` に入れて、使い捨てにする。理由：

1. **セッション状態は揮発性データ**。次のセッションで上書きされる前提。履歴を持つ意味がない
2. **コミットログが汚れる**。毎セッション HANDOVER.md の差分がコミットに混ざる
3. **長期知識は auto-memory に任せる**。VCS 履歴から知識をマイニングする仕組みは、作っても費用対効果が悪い

自分は以前、HANDOVER.md を VCS 管理して差分履歴から知見を抽出するエージェントまで作ったが、結局廃止した。auto-memory が同じ役割を、よりシンプルに果たしてくれる。

## 自動スキル昇格は要らない

「auto-memory に蓄積されたパターンを自動的にスキル（スラッシュコマンド）に昇格する」という仕組みも作って廃止した。

**パターンの検出とパターンの固定化は別の判断。**

- パターンが頻出 ≠ スキル化に値する。一時的な頻出かもしれない
- スキル化するとワークフローが固定される。プロジェクトの方針が変わったとき邪魔になる
- 人間が「これは定型化したい」と思ったときに手動でスキルを作る方が、質が高い

auto-memory に任せて良い:
- パターンの記録（「このPJでは X というアプローチを使う」）
- 知識の蓄積（「Y のデバッグには Z が有効」）

人間が判断すべき:
- スキル化（「このワークフローを固定したい」）
- ルール化（「これを CLAUDE.md に書いて全員に共有したい」）

## 設計方針のまとめ

| 方針 | 具体的にやること |
|------|----------------|
| Claude Code がやることは任せる | auto-memory を ON にして、知識蓄積は Claude に委ねる |
| セッション状態は別で保存する | PreCompact hook で compaction 前に自動保存 |
| 使い捨てを恐れない | HANDOVER.md は .gitignore。VCS 管理しない |
| 自動化の線引きをする | パターンの記録は自動、パターンの固定化は人間 |
| 200行制限を意識する | MEMORY.md はインデックス、詳細はトピックファイルに |

## まとめ

Claude Code の記憶設計で重要なのは「何を覚えるか」ではなく、**「何を Claude に任せて、何を自分で管理するか」の線引き**。

```
Claude に任せる:
  auto-memory — 長期知識の蓄積

自分で管理する:
  CLAUDE.md — プロジェクトルール
  .claude/rules/ — モジュラーなルール

仕組みで補完する:
  PreCompact hook — compaction 前のセッション状態保存
  SessionStart hook — 前セッションの引き継ぎ表示
```

auto-memory は万能ではないが、足りない部分は hook で補完できる。独自の記憶管理を作り込むより、**Claude Code のネイティブ機能を最大限使って、隙間だけ埋める**のが結局一番うまくいく。

公式ドキュメント: [Manage Claude's memory](https://code.claude.com/docs/en/memory) | [Hooks reference](https://code.claude.com/docs/en/hooks)

リポジトリ: https://github.com/kok1eee/o-m-cc
