---
title: "Claude Code の新しい /init を試したら、ついでに隠し環境変数を130個見つけた話"
emoji: "🔍"
type: "tech"
topics: ["ClaudeCode", "AI", "環境変数", "リバースエンジニアリング"]
published: false
---

## TL;DR

Claude Code に**インタビュー形式の新しい `/init`** がテスト中。有効化するには `CLAUDE_CODE_NEW_INIT=1` を設定する。

「この環境変数、どうやって見つけるんだ？」と思ってバイナリを `strings` で調べたら、ドキュメントに載っていない **130個以上の隠しフラグ** が出てきた。

## 新しい /init の発見

きっかけは Thariq（Claude Code チーム）の X でのポスト。

> we're testing a new version of /init based on your feedback- it should interview you and help setup skills, hooks, etc.
> you can enable it with this env_var flag:
> CLAUDE_CODE_NEW_INIT=1 claude

## 有効化

`~/.claude/settings.json` の `env` に追加すれば永続化できる。

```json
{
  "env": {
    "CLAUDE_CODE_NEW_INIT": "1"
  }
}
```

起動時に `CLAUDE_CODE_NEW_INIT=1 claude` でも動くが、毎回打つのは面倒なので settings.json が楽。

## 従来の /init vs 新しい /init

### 従来

1. コードベースをスキャン
2. CLAUDE.md を生成
3. 終わり

### 新しい /init

ステッパー UI で段階的にセットアップが進む。

```
←  ☐ CLAUDE.md  ☐ Skills/Hooks  ✔ Submit  →
```

#### Step 1: CLAUDE.md の種類を選ぶ

```
どの CLAUDE.md ファイルをセットアップしますか？

❯ 1. Project CLAUDE.md
     チームで共有するルール。ソース管理にコミットされる —
     アーキテクチャ、コーディング規約、共通ワークフローなど。
  2. Personal CLAUDE.local.md
     このプロジェクトでの個人的な設定（gitignore対象、非共有）—
     自分の役割、サンドボックスURL、好みのワークフローなど。
  3. Both（両方）
```

**Project CLAUDE.md** と **Personal CLAUDE.local.md** を分けて提案してくれる。`.local.md` は gitignore 対象なので、個人の好みはそちらに書ける。

#### Step 2: Skills + Hooks

```
スキルとフックもセットアップしますか？

❯ 1. Skills + Hooks ✔
  2. Skills only
  3. Hooks only
  4. CLAUDE.md のみ
```

CLAUDE.md だけでなく、Skills（`/deploy` のようなオンデマンドコマンド）と Hooks（ファイル保存後の自動フォーマットなど）も一緒にセットアップできる。

#### Step 3: コードベース調査 + ギャップ質問

Explore エージェントでコードベースをスキャンした後、こう聞いてくる。

> このプロジェクト固有の注意点やワークフローで、コードからは読み取れないものはありますか？

「インタビュー」といっても1問だけなので、深いヒアリングではない。ただ、コードから読み取れない文脈を補完する機会があるのは良い。

#### Step 4: 提案 → 確認 → 生成

最終的にテーブルで提案をまとめて、確認を取ってから生成する。

```
 ☐ 提案

この提案で進めますか？

❯ 1. OK — 進めて
  2. Hookなし
  3. Skillなし
```

## 実際の出力

GAS プロジェクト（yokai_hunt）で試した結果。

| ファイル | 内容 |
|----------|------|
| `CLAUDE.md` | プロジェクト概要、デプロイ手順、アーキテクチャ、制約 |
| `.claude/skills/deploy/SKILL.md` | `/deploy` スキル — `clasp push` でデプロイ |
| `.claude/settings.json` | Format-on-edit フック — `.js` 編集後に prettier 自動実行 |

### Hook のセットアップが丁寧

特に感心したのは Hook の設定プロセス。

1. **環境を検出**: `which prettier` → `mise shims` の `npx prettier` を発見
2. **スコープを限定**: `gas/*.js` のみに prettier を適用（GAS プロジェクトと認識）
3. **パイプテスト**: 実際のファイルパスを流して動作確認
4. **発火テスト**: わざとフォーマット違反を入れて、Hook が自動修正するか検証
5. **JSON 構文チェック**: settings.json の構文・スキーマ検証

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '.tool_response.filePath // .tool_input.file_path' | { read -r f; case \"$f\" in */gas/*.js) ~/.local/share/mise/shims/npx prettier --write \"$f\";; esac; } 2>/dev/null || true"
          }
        ]
      }
    ]
  }
}
```

「Hook を作って終わり」ではなく、**動作確認まで自動でやる**のは信頼できる。

### 改善点

- **既存設定との重複チェックがない**: グローバルの `~/.claude/settings.json` に既に prettier Hook がある場合でも、プロジェクトレベルで重複して作ってしまう
- **プラグイン検出がない**: インストール済みプラグインのスキル・ワークフローを考慮しない。手動で「o-m-cc を考慮して」と伝える必要があった
- **インタビューが浅い**: 1問だけ。プロジェクトの運用体制やチーム構成まで聞いてくれるともっと精度が上がりそう
- **所要時間**: 約5分。従来の `/init` は数秒だったので体感は重い

## ここからが本題: 隠し環境変数の探し方

`CLAUDE_CODE_NEW_INIT` を settings.json に追加した時点でふと思った。**こういうフラグ、他にどれくらいあるんだ？**

Claude Code は Bun でコンパイルされたネイティブバイナリだが、文字列リテラルはそのまま埋まっている。つまり：

```bash
strings $(which claude) | grep -oE 'CLAUDE_CODE_[A-Z_]+' | sort -u
```

これで全部抜ける。結果、**130個以上** の `CLAUDE_CODE_` プレフィックス付き環境変数が見つかった。

130個全部リストアップしても使い物にならないので、実用的なものを厳選する。

## 厳選: 使える隠しフラグ

### CLAUDE_CODE_SUBAGENT_MODEL

```json
"CLAUDE_CODE_SUBAGENT_MODEL": "sonnet"
```

サブエージェント（Agent ツールで起動される子プロセス）のモデルを強制指定する。選択肢は `"sonnet"`, `"haiku"`, `"opus"`, `"inherit"`。

バイナリからコードを読むと：

```js
if (process.env.CLAUDE_CODE_SUBAGENT_MODEL)
  return $K(process.env.CLAUDE_CODE_SUBAGENT_MODEL);
```

一番最初にチェックされるので、他のどの設定よりも優先される。

**使いどころ**: メインは Opus で回しつつ、サブエージェントは Sonnet にしてコスト削減。調査系エージェントに Opus は過剰なことが多い。個人的に一番使えるフラグ。

### CLAUDE_CODE_GLOB_HIDDEN / CLAUDE_CODE_GLOB_NO_IGNORE

```json
"CLAUDE_CODE_GLOB_HIDDEN": "1",
"CLAUDE_CODE_GLOB_NO_IGNORE": "1"
```

- `GLOB_HIDDEN`: Glob ツールで `.` で始まる隠しファイルも検索対象に
- `GLOB_NO_IGNORE`: `.gitignore` に含まれるファイルも検索対象に

設定ファイルやビルド成果物を調査したいとき、「Claude がファイルを見つけられない」問題がこれで解決することがある。

### CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD

別ディレクトリの CLAUDE.md も読み込ませる。モノレポで共通の CLAUDE.md を参照したいときに。

### CLAUDE_CODE_FILE_READ_MAX_OUTPUT_TOKENS

ファイル読み取り時のトークン上限を変更。大きなファイルを一度に読みたいときに。

### 動作制御系

| フラグ | 効果 |
|--------|------|
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY` | 自動メモリ保存を無効化 |
| `CLAUDE_CODE_DISABLE_FAST_MODE` | Fast モード無効化 |
| `CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING` | Adaptive thinking を常にオフ |
| `CLAUDE_CODE_DISABLE_TERMINAL_TITLE` | ターミナルタイトルの書き換えを無効化 |

## おまけ: コンパクトは遅らせられない

コンテキストウィンドウが一杯になると自動でコンパクト（要約）が走る。このタイミングを調整するフラグも見つかった。

| フラグ | 効果 |
|--------|------|
| `CLAUDE_CODE_AUTO_COMPACT_WINDOW` | コンパクト発火のトークン数上限 |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` | コンテキストの何%で発火するか |

「長く使いたいから上限を上げよう」と思ったが、バイナリのコードを読むと **下げる方向にしか効かない**。

```js
// バイナリから抜粋（整形済み）
q = Math.min(q, R)                       // AUTO_COMPACT_WINDOW
let $ = Math.floor(T * (R / 100));
return Math.min($, q)                     // AUTOCOMPACT_PCT_OVERRIDE
```

どちらも `Math.min` で「デフォルトか指定値の小さい方」を採用する設計。つまり**コンパクトを遅らせたいなら、何も設定しないのが最適解**。

## まとめ

```bash
# これだけ覚えておけばいい
strings $(which claude) | grep -oE 'CLAUDE_CODE_[A-Z_]+' | sort -u
```

新しい `/init` は「CLAUDE.md + Skills + Hooks を対話的にセットアップ」してくれる明確な進化。ただしまだテスト中で、プラグイン検出や既存設定との重複チェックには改善の余地がある。

隠しフラグは130個以上あるが、全部覚える必要はない。探し方さえ知っていれば十分。
