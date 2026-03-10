---
title: "wez-sidebar v0.2 — Claude Code並列セッション監視TUIを最小構成に作り直した"
emoji: "🪓"
type: "tech"
topics: ["claudecode", "wezterm", "rust", "tui"]
published: false
---

:::message
**TL;DR** — Claude Code 並列セッション監視ツール「wez-sidebar」を v0.2 にリニューアルした。自分専用の機能を切り離し、WezTerm ユーザーなら誰でも使える最小構成に作り直した。
:::

## はじめに

> 業務自動化 Python エンジニア。バイブコーディング歴 1 年 ≒ エンジニア歴。

以前、v0.1 として wez-sidebar を公開した。

https://zenn.dev/kok1eee/articles/claude-code-parallel-wezterm-wez-sidebar

その後、cmux が登場した。

https://github.com/manaflow-ai/cmux

cmux は Ghosty ユーザー向けに設計された Claude Code 監視ツールで、UI の完成度と設定の細かさでは正直勝てない。wez-sidebar は WezTerm のペイン内 TUI という制約の中で動くので、cmux のようなリッチな実装はそもそも難しい。

ただ「WezTerm ユーザーが同一ウィンドウ内で完結させる」という軸なら話は別だと思った。

そのために v0.1 で自分の ambient-task-agent と密結合させていた部分を全部切り離して、WezTerm + Claude Code の組み合わせさえあれば動く最小構成に作り直した。

これは OSS 的なものとしては sdtab に続いて 2 本目。汎用ツールを作ったというよりも、「自分が使うツールを公開したので参考にして改造してほしい」というスタンス。

https://github.com/kok1eee/wez-sidebar

## なにができるか

WezTerm のウィンドウ内にサイドバーまたはドックとして常駐し、Claude Code のセッション状態をリアルタイムで監視する。

<!-- TODO: MacBook スクショ -->
<!-- ![MacBook: 複数ペイン + 右サイドバー]() -->

<!-- TODO: 外部モニター スクショ -->
<!-- ![外部モニター: 複数ペイン + 下部ドック]() -->

### セッション監視

各セッションのカードに以下が表示される：

```
╭─ 🟢 my-project ⠋ ────╮
│ 2h30m  main           │
│ Edit src/config.rs     │
│ fix the bug (3m ago)   │
╰───────────────────────╯
```

- **ステータス：** 実行中（スピナー）/ 入力待ち（?）/ 停止（■）
- **稼働時間 + git ブランチ**
- **現在のアクティビティ：** `Edit src/config.rs`、`Bash cargo test` など、今何をやっているかをリアルタイム表示
- **最後のユーザーメッセージ：** 何を指示したかと経過時間

### 危険コマンド警告

`rm -rf` や `git push --force` などを Claude Code が実行しようとすると、カードが赤くなって ⚠ マーカーが付く。yolo モード（`--dangerously-skip-permissions`）で動かしているセッションは 🤖 マークで区別される。

### API 使用量モニター

Anthropic の使用量 API（OAuth 経由）から 5 時間制限・週間制限を取得して表示。

- 緑：余裕あり / 黄：50% 超 / 赤：80% 超
- リセットまでの残り時間も表示

### ペイン切り替え

Enter または数字キーで選択したセッションのペインに即ジャンプ。

## v0.1 からの変化

v0.1 は ambient-task-agent との連携を前提に設計していた。タスク同期、hook の外部委譲など、自分の環境に特化した機能が混在していた。

v0.2 ではそれを全部外した。

| | v0.1 | v0.2 |
|--|------|------|
| タスク表示 | Asana 連携（外部 JSON） | TaskCreate/TaskUpdate + TodoWrite 連携 |
| hook 処理 | 外部委譲可能 | 内蔵のみ |
| 設定項目 | `tasks_file`, `hook_command` 等 | 最小限（3 項目） |
| インストール | ソースビルドのみ | バイナリ配布あり |

狭いスペースに詰め込むのではなく、本当に必要なものを広く使えるように、というのが基本方針。

## アーキテクチャ

```
Claude Code ──hook──→ wez-sidebar hook <event>
                              │
                    ┌─────────┴──────────┐
                    │ セッション状態更新  │
                    │ アクティビティ抽出 │
                    │ 危険コマンド検出   │
                    │ Yoloモード検出     │
                    │ gitブランチ取得    │
                    └─────────┬──────────┘
                              │
                    sessions.json
                              │
                         file watcher
                              │
                    wez-sidebar TUI（ゼロポーリング）
```

hook イベントを受け取るたびに `sessions.json` を更新し、TUI 側はファイル変更を検知して表示を更新する。ポーリングなし。

hook ハンドラーは内蔵なので外部依存ゼロ。`wez-sidebar hook PreToolUse` を呼ぶだけで動く。

## セットアップ

### インストール

バイナリ配布あり（Rust ツールチェーン不要）：

```bash
# macOS (Apple Silicon)
curl -L https://github.com/kok1eee/wez-sidebar/releases/latest/download/wez-sidebar-aarch64-apple-darwin \
  -o ~/.local/bin/wez-sidebar && chmod +x ~/.local/bin/wez-sidebar
```

ソースから：

```bash
cargo install wez-sidebar
# または
git clone https://github.com/kok1eee/wez-sidebar.git
cd wez-sidebar && cargo install --path .
```

初回セットアップはウィザードが案内してくれる：

```bash
wez-sidebar init
```

### Claude Code hooks 登録

`~/.claude/settings.json` に追加：

```json
{
  "hooks": {
    "PreToolUse": [
      { "type": "command", "command": "wez-sidebar hook PreToolUse" }
    ],
    "PostToolUse": [
      { "type": "command", "command": "wez-sidebar hook PostToolUse" }
    ],
    "Notification": [
      { "type": "command", "command": "wez-sidebar hook Notification" }
    ],
    "Stop": [
      { "type": "command", "command": "wez-sidebar hook Stop" }
    ],
    "UserPromptSubmit": [
      { "type": "command", "command": "wez-sidebar hook UserPromptSubmit" }
    ]
  }
}
```

### WezTerm キーバインド

```lua
-- wezterm.lua
config.keys = {
  -- Sidebar（MacBook 向け）
  {
    key = "b",
    mods = "LEADER",
    action = wezterm.action_callback(function(window, pane)
      pane:split({ direction = "Right", size = 0.2, args = { "wez-sidebar" } })
    end),
  },
  -- Dock（外部モニター向け）
  {
    key = "d",
    mods = "LEADER",
    action = wezterm.action_callback(function(window, pane)
      pane:split({ direction = "Bottom", size = 0.25, args = { "wez-sidebar", "dock" } })
    end),
  },
}
```

設定ファイルは不要。これだけで動く。

## キーバインド

| キー | 動作 |
|------|------|
| `j`/`k` | セッション移動 |
| `Enter` | ペインに切り替え |
| `1`-`9` | 番号で直接切り替え |
| `p` | ペインプレビュー表示 |
| `f` | stale セッション表示切り替え |
| `d` | セッション削除 |
| `r` | 手動リフレッシュ |
| `?` | ヘルプ |

## おわりに

cmux と比べて UI が地味なのは認める。WezTerm の TUI という制約上、リッチな実装は難しい。

ただ WezTerm を使っていて、Claude Code を並列で動かしていて、同じウィンドウ内で完結させたいなら、これが一番シンプルに動くと思う。

コードは MIT ライセンスで公開している。自分の環境に合わせて改造して使ってほしい。

https://github.com/kok1eee/wez-sidebar
