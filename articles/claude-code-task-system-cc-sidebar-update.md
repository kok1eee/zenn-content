---
title: "Claude Code 2.1.16のTask System対応 - o-m-ccとcc-sidebarをアップデートした"
emoji: "🔄"
type: "tech"
topics: ["claudecode", "ai", "cli", "wezterm"]
published: false
---

## はじめに

> 業務自動化Pythonエンジニア。バイブコーディング歴1年 ≒ エンジニア歴。

Claude Code 2.1.16で**TaskCreate/TaskUpdate**という新しいタスク管理システムが入った。従来のTodoWriteを置き換えるもので、依存関係トラッキングが組み込まれている。

これに合わせて、自作のプラグイン「o-m-cc」とサイドバーツール「cc-sidebar」を全面的にアップデートした話。

## Claude Code 2.1.16のTask System

### TodoWriteとの違い

| 機能 | TodoWrite（旧） | TaskCreate/TaskUpdate（新） |
|------|-----------------|---------------------------|
| タスク作成 | TodoWrite | TaskCreate |
| ステータス更新 | TodoWrite（上書き） | TaskUpdate(taskId, status) |
| 依存関係 | なし | addBlocks / addBlockedBy |
| 一覧取得 | なし | TaskList |
| 詳細取得 | なし | TaskGet |
| ストレージ | ファイルベース | セッション内メモリ |

### 依存関係が嬉しい

```
TaskCreate: "DB設計"         → #1
TaskCreate: "API実装"        → #2
TaskUpdate: taskId=2, addBlockedBy=["1"]
```

これで「DB設計が終わるまでAPI実装に着手しない」という制約をエージェントが自動で守る。

### 注意点

- TaskCreate/TaskUpdateは**セッションスコープ**（セッション終了で消える）
- Hookからは呼べない（AIエージェント専用）
- `CLAUDE_CODE_ENABLE_TASKS=false`で旧TodoWriteに戻せる

## o-m-cc v0.7.3

https://github.com/kok1eee/o-m-cc

### 変更点

#### TodoWrite完全削除

全エージェント・コマンド・テンプレートからTodoWriteを排除。TaskCreate/TaskUpdateに統一。

```diff
# agents/planner.md
- tools: Read, Glob, Grep, Write, TodoWrite
+ tools: Read, Glob, Grep, Write, TaskCreate, TaskUpdate
```

#### sync-tasks.sh（新Hook）

TaskCreate/TaskUpdateのPostToolUseフックで、`spec/plan/tasks.md`にリアルタイム同期。

```bash
# TaskCreate → tasks.mdに追記
- [ ] DB設計 <!-- task:1 -->

# TaskUpdate(in_progress) → activeマーク
- [ ] API実装 <!-- task:2 status:active -->

# TaskUpdate(completed) → チェック
- [x] DB設計 <!-- task:1 -->
```

#### reset-tasks.sh（SessionStart Hook）

セッション開始時にカウンターをリセット。IDの衝突を自動防止。

```
SessionStart
  → カウンターリセット
  → 古いtask IDコメント削除
  → tasks.mdのチェックボックス状態は保持
```

### フロー図

```
TaskCreate("タスク名")
  → sync-tasks.sh (PostToolUse hook)
  → spec/plan/tasks.md に追記
  → cc-sidebar が fsnotify で検知
  → リアルタイム表示更新
```

## cc-sidebar

### 概要（おさらい）

WezTermのペインにClaude Codeのセッション状況を常時表示するTUIツール。Go + Bubble Tea製。

```
⚡ cc-sidebar

📊 Usage
🕐 17:50:53
⏳ 5h: 9% (2h56m)
📅 All: 30% (火19:59)
💬 Sonnet: 1%

🖥  Sessions [All]
1 ● my-project 1/3(33%) ▶
  API実装
2 ○ other-project ?
```

### 今回のアップデート

#### 1. tasks.md直接監視

以前は中間ファイル（tasks.json）をHookで生成していたが、cc-sidebarが直接`spec/plan/tasks.md`を監視する方式に変更。

```
旧: TaskCreate → Hook → tasks.json → cc-sidebar
新: TaskCreate → Hook → tasks.md → cc-sidebar (fsnotify)
```

中間ファイルが不要になり、構成がシンプルに。

#### 2. フラット形式対応

Phase構造（Phase 1, Phase 2...）への依存を廃止。チェックボックスの数だけカウント。

```markdown
- [x] DB設計 <!-- task:1 -->
- [ ] API実装 <!-- task:2 status:active -->
- [ ] テスト <!-- task:3 -->
```

cc-sidebar表示: `1/3(33%)` + 現在のタスク名

#### 3. 2行レイアウト

セッションごとに2行で表示：

```
1 ● project-name 1/3(33%) ▶    ← プロジェクト名 + 進捗
  API実装                        ← 現在作業中のタスク
```

横幅が狭いサイドバーでも見やすい。

#### 4. 同一ウィンドウフィルタ

`WEZTERM_PANE`環境変数でcc-sidebar自身のウィンドウを特定。同じウィンドウ内のセッションのみ表示。別ウィンドウのセッションはペイン切り替えできないので非表示に。

## アーキテクチャ全体像

```
┌─────────────────────────────────────────────────┐
│ Claude Code Session                              │
│                                                  │
│  TaskCreate("タスク名")                          │
│       ↓ PostToolUse                              │
│  sync-tasks.sh → spec/plan/tasks.md              │
│                                                  │
│  TaskUpdate(completed)                           │
│       ↓ PostToolUse                              │
│  sync-tasks.sh → tasks.md の [x] 更新           │
└──────────────────────────────────────────────────┘
         ↓ fsnotify (ファイル変更検知)
┌──────────────────────────────────────────────────┐
│ cc-sidebar (Go + Bubble Tea)                     │
│                                                  │
│  tasks.go: parseTasksMd() でチェックボックス集計  │
│  main.go: View() で 2行レイアウト表示             │
└──────────────────────────────────────────────────┘
```

## ビルド時の注意（macOS）

Go製バイナリをmacOSで使う場合、リビルド後にアドホック署名が必要：

```bash
go build -o cc-sidebar .
codesign -s - cc-sidebar
cp cc-sidebar ~/.local/bin/
```

署名しないとmacOSがバイナリの実行をブロックする。

## まとめ

- Claude Code 2.1.16のTaskCreate/TaskUpdateは依存関係付きのタスク管理
- o-m-ccは完全にTask Systemに移行（TodoWrite廃止）
- cc-sidebarはtasks.mdを直接監視してリアルタイム進捗表示
- Hookで自動同期するのでユーザーは意識する必要なし

「エージェントがTaskCreate呼ぶだけでサイドバーに進捗が出る」という体験がシンプルで良い。
