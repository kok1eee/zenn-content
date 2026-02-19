---
title: "Claude Code並列セッションをWezTermで快適に管理する - wez-sidebar"
emoji: "🖥️"
type: "tech"
topics: ["claudecode", "wezterm", "rust", "terminal"]
published: false
---

## はじめに

> 業務自動化Pythonエンジニア。バイブコーディング歴1年 ≒ エンジニア歴。

以前、claude-code-monitorに感動してWezTermをメインにした話を書いた。

https://zenn.dev/kok1eee/articles/claude-code-monitor-wezterm-contribution

その後、「IDEでClaude Code動かすな」という海外トレンドに乗っかって、Zed + ターミナル派の話も書いた。

https://zenn.dev/kok1eee/articles/claude-code-zed-terminal-workflow

WezTermに移行してからは並列セッション数がさらに増えた。4〜6セッション同時に走らせていると、ccmのセッション監視だけでは足りなくなってきた。

- API使用量、あとどれくらい余裕ある？
- タスクの進捗を一目で把握したい
- WezTermの**同じウィンドウ内**で完結させたい（別ウィンドウじゃなく）

ccmは素晴らしいツールだけど、自分のワークフローに合わせてカスタマイズしたくなった。

そこで**wez-sidebar**を作った。WezTermのサイドバーまたはドックとして常駐し、Claude Codeの状態をリアルタイムで監視する。

https://github.com/kok1eee/wez-sidebar

## 全体像

WezTermのウィンドウ内に複数のClaude Codeセッション + wez-sidebarを配置する構成。

![Dock mode](/images/wez-sidebar-dock.png)
*Dock mode: 画面下部に3カラムで配置*

![Sidebar mode](/images/wez-sidebar-sidebar.png)
*Sidebar mode: 右端にサイドバーとして配置*

## 機能

### セッション監視

Claude Codeのhookを使って、各セッションの状態をリアルタイムで追跡する。

| 表示 | 意味 |
|------|------|
| ▶ | 実行中（ツール呼び出し中） |
| ? | 入力待ち（permission prompt） |
| ■ | 停止済み |

セッションごとに稼働時間、タスク進捗（TaskCreate/TaskUpdate連携）も表示される。

### API使用量モニター

Anthropicの使用量API（OAuth経由）から5時間制限・週間制限をリアルタイム取得。

- 緑: 余裕あり
- 黄: 50%超
- 赤: 80%超（そろそろ止まる）

リセットまでの残り時間も表示されるので、「あとどれくらい使えるか」が一目でわかる。

### ペイン切り替え

`Enter`キーまたは数字キーで、選択したセッションのWezTermペインに即ジャンプ。4分割で動かしていても迷わない。

### タスク表示（オプション）

外部のタスクキャッシュJSONファイルを読み込んで、タスク一覧を表示できる。Asana連携などで使う場合に設定する。

## 2つの表示モード

### Sidebar mode

```bash
wez-sidebar
```

WezTermのペイン分割で右端に細く配置する想定。Usage → Tasks → Sessionsが縦に並ぶ。

### Dock mode

```bash
wez-sidebar dock
```

画面下部に横長で配置する想定。Usage・Tasks・Sessionsが3カラムで横に並ぶ。

## WezTermとの統合

### Overlay で起動する方法

WezTermの`key_table`を使って、overlayとして起動するのがおすすめ。

![WezTerm overlay](/images/wez-sidebar-overlay.png)
*overlayとして起動すれば、ペインを分割せずに済む*

```lua
-- wezterm.lua
config.keys = {
  {
    key = "s",
    mods = "LEADER",
    action = wezterm.action.SpawnCommandInNewTab({
      args = { "wez-sidebar" },
    }),
  },
}
```

もちろんペイン分割で常駐させるのもあり。

### サイドバーとして常駐

```lua
{
  key = "b",
  mods = "LEADER",
  action = wezterm.action_callback(function(window, pane)
    pane:split({ direction = "Right", size = 0.2, args = { "wez-sidebar" } })
  end),
}
```

### ドックとして常駐

```lua
{
  key = "d",
  mods = "LEADER",
  action = wezterm.action_callback(function(window, pane)
    pane:split({ direction = "Bottom", size = 0.25, args = { "wez-sidebar", "dock" } })
  end),
}
```

## セットアップ

### 1. インストール

```bash
git clone https://github.com/kok1eee/wez-sidebar.git
cd wez-sidebar
cargo install --path .
```

### 2. Claude Code hooks 登録

`~/.claude/settings.json` に追加:

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

hookが呼ばれるたびに`sessions.json`が更新され、wez-sidebarがファイルウォッチで即反映する。

### 3. 設定（オプション）

```bash
mkdir -p ~/.config/wez-sidebar
cp config.example.toml ~/.config/wez-sidebar/config.toml
```

```toml
# タスク表示を有効にする場合
tasks_file = "~/.config/wez-sidebar/tasks-cache.json"
task_filter_name = "Your Name"

# hookを外部コマンドに委譲する場合
# hook_command = "ambient-task-agent hook"
```

## 仕組み

```
Claude Code (session A) ──hook──→ wez-sidebar hook PreToolUse
Claude Code (session B) ──hook──→ wez-sidebar hook Notification
                                        │
                                        ▼
                                  sessions.json
                                        │
                                   file watcher
                                        │
                                        ▼
                                  wez-sidebar TUI
```

1. Claude Codeがhookイベントを発火
2. `wez-sidebar hook <event>` がstdinからセッション情報を受け取る
3. 親プロセスのTTYを検出し、sessions.jsonを更新
4. TUI側がファイル変更を検知して表示を更新

hookハンドラーは内蔵しているので、wez-sidebar単体で動く。外部ツールに委譲したい場合は`hook_command`で設定可能。

## まとめ

Claude Codeの並列セッション運用には、セッション状態の可視化とペイン切り替えが必須。wez-sidebarをWezTermに組み込むことで、複数セッションの管理が格段に楽になった。

- セッション状態がリアルタイムでわかる
- API使用量を常に把握できる
- ペイン切り替えがワンキー

https://github.com/kok1eee/wez-sidebar
