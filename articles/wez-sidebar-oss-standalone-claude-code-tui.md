---
title: "Claude Codeの並列セッション、どう管理する？ — 自分なりの答え wez-sidebar v0.2"
emoji: "🪓"
type: "tech"
topics: ["claudecode", "wezterm", "rust", "tui"]
published: true
---

:::message
**TL;DR** — WezTerm + Claude Code環境向けのセッション監視TUI「wez-sidebar」を作った。cmuxには機能で勝てないが、WezTermの同一ウィンドウ内で完結することに振り切っている。
:::

## はじめに

> 業務自動化エンジニア。バイブコーディング歴1年 ≒ エンジニア歴。

Claude Codeを並列で4〜6セッション同時に動かすようになってから、「どのセッションが今何をやっているか」「APIの残量はどのくらいか」を把握するのが地味にしんどくなった。

そこでWezTermのサイドバーまたはドックとして常駐する監視TUI、**wez-sidebar**を作った。

https://github.com/kok1eee/wez-sidebar

ちなみに最近[cmux](https://github.com/mskelton/cmux)というClaude Code監視ツールも登場している。UIの完成度と設定の細かさでは正直勝てない。wez-sidebarはWezTermのペイン内TUIという制約の中で動くので、cmuxのようなリッチな実装はそもそも難しい。

ただwez-sidebarには「WezTermの同一ウィンドウ内で完結する」という軸がある。cmuxはGhosty向けに設計されており、別ウィンドウとして動く。WezTermを使っていて、ペイン切り替えをターミナル内で完結させたい人には、こちらの方が合うと思っている。

これはOSSとしてはsdtabに続いて2本目。汎用ツールというより「自分が使うものを公開したので参考にして改造してほしい」というスタンス。

## 画面

![MacBook sidebar](https://raw.githubusercontent.com/kok1eee/wez-sidebar/main/docs/images/sidebar-with-panes.png)
*MacBook: 複数ペイン + 右サイドバー*

![Dock mode](https://raw.githubusercontent.com/kok1eee/wez-sidebar/main/docs/images/dock-mode.png)
*外部モニター: 複数ペイン + 下部ドック*

## 機能

### セッションカード

各Claude Codeセッションがカード形式で表示される。

**Sidebar（コンパクト）**
```
╭─ 🟢 my-project ⠋ ────╮
│ 2h30m  main           │
│ Edit src/config.rs     │
│ fix the bug (3m ago)   │
╰───────────────────────╯
```

**Dock（タスク進捗付き）**
```
╭─ 🟢 my-project ⠋ ─────────────╮
│ 2h30m  main                    │
│ Edit src/hooks.rs              │
│ implement auth (5m ago)        │
│ ✓ Add types                    │
│ ● Edit hooks                   │
│ ○ Add tests                    │
╰────────────────────────────────╯
```

カードに含まれる情報：

- **ステータス**: ⠋ 実行中 / `?` 入力待ち / `■` 停止
- **稼働時間 + gitブランチ**
- **現在のアクティビティ**: `Edit src/config.rs`、`Bash cargo test` など今何をやっているかをリアルタイム表示
- **最後のユーザーメッセージ + 経過時間**
- **サブエージェント数**: `claude --agent`で起動したサブエージェントが動いていれば `2agents` のように表示

セッションマーカー：

| マーカー | 意味 |
|--------|------|
| 🟢 | 現在のペイン |
| 🔵 | 他のペイン |
| 🤖 | Yoloモード（`--dangerously-skip-permissions`） |
| ⚫ | 切断済み（ペインが閉じられた状態、24時間保持） |

### 危険コマンド警告

`rm -rf`、`git push --force`、`DROP TABLE`などをClaude Codeが実行しようとすると、カードが赤くなって⚠マーカーが付く。yoloモードで動いているセッションは🤖マークで区別される。

### API使用量モニター

Anthropicの使用量API（OAuth経由）から5時間制限・週間制限をリアルタイム取得。緑→黄→赤でリセットまでの残り時間とセットで表示される。「あとどれくらい使えるか」が常に見える。

### ペイン切り替え

`Enter`または数字キーで選択したセッションのWezTermペインに即ジャンプ。4〜6分割で動かしていても迷わない。

### 孤児プロセスのクリーンアップ（オプトイン）

Claude Codeのセッションを途中で強制終了したりWezTermのペインを閉じると、裏でclaudeプロセスが残り続けることがある。reaperを有効にすると、WezTermのどのペインにも紐づいていないclaudeプロセスを定期的に検出してSIGTERMで終了させる。

```toml
# ~/.config/wez-sidebar/config.toml
[reaper]
enabled = true
threshold_hours = 3  # 3時間以上放置された孤児を対象
```

TUIが5分おきに自動実行するほか、手動でも確認できる：

```bash
wez-sidebar reap --dry  # killせずに対象プロセスを一覧表示
wez-sidebar reap        # SIGTERM で終了
```

## 仕組み

```
Claude Code ──hook──→ wez-sidebar hook <event>
                              │
                    ┌─────────┴──────────────┐
                    │ セッション状態更新      │
                    │ アクティビティ抽出      │
                    │ 危険コマンド検出        │
                    │ サブエージェント追跡    │
                    │ gitブランチ取得         │
                    │ Yoloモード検出          │
                    └─────────┬──────────────┘
                              │
                    sessions.json / usage-cache.json
                              │
                         file watcher
                              │
                    wez-sidebar TUI（ゼロポーリング）
                              │
                    reaper（オプトイン、5分間隔）
                    └→ ps + wezterm cli list → 孤児kill
```

hookイベントを受け取るたびに`sessions.json`を更新し、TUI側はファイル変更を検知して表示を更新する。ポーリングなし、外部依存なし。

## セットアップ

### インストール

Rustなしでバイナリをそのまま使える：

```bash
# macOS (Apple Silicon)
curl -L https://github.com/kok1eee/wez-sidebar/releases/latest/download/wez-sidebar-aarch64-apple-darwin \
  -o ~/.local/bin/wez-sidebar && chmod +x ~/.local/bin/wez-sidebar

# macOS (Intel)
curl -L https://github.com/kok1eee/wez-sidebar/releases/latest/download/wez-sidebar-x86_64-apple-darwin \
  -o ~/.local/bin/wez-sidebar && chmod +x ~/.local/bin/wez-sidebar

# Linux (x86_64)
curl -L https://github.com/kok1eee/wez-sidebar/releases/latest/download/wez-sidebar-x86_64-linux \
  -o ~/.local/bin/wez-sidebar && chmod +x ~/.local/bin/wez-sidebar
```

Rustがある場合：

```bash
cargo install wez-sidebar
```

### セットアップウィザード

```bash
wez-sidebar init
```

hooksの登録とWezTermキーバインドの例を案内してくれる。

### 手動セットアップ

`~/.claude/settings.json`にhooksを追加：

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

### WezTermレイアウト

wez-sidebarはWezTermのペインとして動くので、セッション用ペインと一緒にレイアウトを組む。自分は2パターンを使い分けている。

**Dockモード（外部モニター向け）** — 3×2 + 下部ドック：

```
+--------+--------+--------+
|   1    |   3    |   5    |
+--------+--------+--------+
|   2    |   4    |   6    |
+--------+--------+--------+
|         wez-sidebar dock  |
+---------------------------+
```

**Sidebarモード（MacBook向け）** — 2×2 + 右サイドバー：

```
+--------+--------+------+
|   1    |   3    |      |
+--------+--------+ side |
|   2    |   4    | bar  |
+--------+--------+------+
```

`wezterm.lua` で起動時に自動構築する例：

```lua
local wezterm = require("wezterm")
local config = wezterm.config_builder()

-- 起動時レイアウト（Dockモード: 3x2 + dock）
wezterm.on("gui-startup", function(cmd)
  local home = os.getenv("HOME")
  local tab, pane1, window = wezterm.mux.spawn_window({})

  -- 下部にdockを作成（25%高さ）
  pane1:split({ direction = "Bottom", size = 0.25,
    args = { home .. "/.local/bin/wez-sidebar", "dock" } })

  -- 上部を上下に分割
  local pane2 = pane1:split({ direction = "Bottom", size = 0.5 })

  -- 上段を3分割
  local pane3 = pane1:split({ direction = "Right", size = 0.67 })
  local pane5 = pane3:split({ direction = "Right", size = 0.5 })

  -- 下段を3分割
  local pane4 = pane2:split({ direction = "Right", size = 0.67 })
  local pane6 = pane4:split({ direction = "Right", size = 0.5 })

  pane1:activate()
end)
```

キーバインドで新しいタブとしてレイアウトを作る場合：

```lua
config.leader = { key = "a", mods = "CTRL", timeout_milliseconds = 2000 }

config.keys = {
  -- Leader+t でレイアウト選択して新規タブ作成
  { key = "t", mods = "LEADER", action = wezterm.action.InputSelector({
    title = "New Tab Layout",
    choices = {
      { id = "dock", label = "Dock (3x2 + bottom bar)" },
      { id = "sidebar", label = "Sidebar (2x2 + right bar)" },
    },
    action = wezterm.action_callback(function(window, pane, id, label)
      local home = os.getenv("HOME")
      local tab, pane1, _ = window:mux_window():spawn_tab({})
      if id == "dock" then
        -- 3x2 + dock
        pane1:split({ direction = "Bottom", size = 0.25,
          args = { home .. "/.local/bin/wez-sidebar", "dock" } })
        local pane2 = pane1:split({ direction = "Bottom", size = 0.5 })
        local pane3 = pane1:split({ direction = "Right", size = 0.67 })
        local pane5 = pane3:split({ direction = "Right", size = 0.5 })
        local pane4 = pane2:split({ direction = "Right", size = 0.67 })
        local pane6 = pane4:split({ direction = "Right", size = 0.5 })
      else
        -- 2x2 + sidebar
        pane1:split({ direction = "Right", size = 0.12,
          args = { home .. "/.local/bin/wez-sidebar" } })
        local pane2 = pane1:split({ direction = "Bottom", size = 0.5 })
        local pane3 = pane1:split({ direction = "Right", size = 0.5 })
        local pane4 = pane2:split({ direction = "Right", size = 0.5 })
      end
      pane1:activate()
    end),
  }) },
}
```

設定ファイルは不要。hooks + WezTermレイアウトだけで動く。

## おわりに

cmuxと比べてUIが地味なのは認める。WezTermのペイン内TUIという制約上、できることに限界がある。

ただWezTermを使っていて、Claude Codeを並列で動かしていて、同じウィンドウ内で完結させたいなら、これが一番シンプルに動くと思う。MITライセンスで公開しているので、自分の環境に合わせて改造して使ってほしい。

https://github.com/kok1eee/wez-sidebar
