---
title: "Claude Code複数セッション監視ツールが便利すぎてWezTermがメインになりそう"
emoji: "🖥️"
type: "tech"
topics: ["claudecode", "wezterm", "zed", "terminal"]
published: false
---

## はじめに

> 業務自動化Pythonエンジニア。バイブコーディング歴1年 ≒ エンジニア歴。

Claude Codeを複数セッション並行で動かしていると、「今どのセッションが動いてる？」「あのセッションにフォーカス移動したい」という場面が増えてきた。

WezTermとZedを両方使っていて、メインはZedだった。でも今回見つけたツールのおかげで、WezTermがメインになりそう。

## claude-code-monitorがめっちゃいい

https://github.com/onikan27/claude-code-monitor

複数のClaude Codeセッションをリアルタイムで監視するCLIダッシュボード。

```
╭──────────────────────────────────────────────────────────╮
│ Claude Code Monitor │ ● 1 ◐ 1 ✓ 2                        │
╰──────────────────────────────────────────────────────────╯
│  [1] ● Running   ~/project-a                             │
│  [2] ◐ Waiting   ~/project-b                             │
│  [3] ✓ Done      ~/project-c                             │
╰──────────────────────────────────────────────────────────╯
         [↑↓]Select [Enter]Focus [1-9]Quick [q]Quit
```

### 何がいいか

| 機能 | 説明 |
|------|------|
| セッション監視 | 複数セッションの状態をリアルタイム表示 |
| ステータス表示 | ● 実行中 / ◐ 入力待機 / ✓ 完了 |
| **フォーカス切替** | 選択したターミナルタブに即ジャンプ |

**フォーカス切替が最高**。番号押すだけで該当セッションのタブに飛べる。

4分割でClaude Code動かしていると、どのペインがどのプロジェクトかわからなくなる。ccm watchを別ウィンドウで開いておけば、一発でジャンプできる。

## 問題：Zedの内蔵ターミナルでは使えない

Zedの内蔵ターミナルでClaude Code動かしていたが、**ccmのフォーカス切替が動かない**。

理由：
- ccmはiTerm2、Terminal.app、WezTermなどのターミナルアプリに対応
- Zedの内蔵ターミナルは外部から制御するAPIがない
- フォーカス切替には外部からタブ/ペインを操作する必要がある

| ターミナル | フォーカス切替 |
|-----------|--------------|
| iTerm2 | ✓ AppleScript |
| WezTerm | ✓ wezterm cli |
| Zed Terminal | ✗ 制御API無し |
| VSCode Terminal | ✗ Extension APIのみ |

## WezTermがメインになりそう

ccmのフォーカス切替を使いたいので、Claude CodeはWezTermで動かすことにした。

### 構成

```
WezTerm
├── ペイン1: Claude Code (project-a)
├── ペイン2: Claude Code (project-b)
├── ペイン3: Claude Code (project-c)
└── ペイン4: ccm watch
```

ccmでどのペインにも一発ジャンプ。Zedはコード確認用として併用。

### LazyVimも頑張ってみる

WezTermをメインにするなら、エディタもターミナル内で完結させたい。

LazyVimはNeovimのディストリビューション。設定済みの状態から始められるので、Neovim初心者でも入りやすい。

Zedと併用しつつ、徐々にLazyVimに慣れていく予定。

## WezTerm対応について

最初はWezTermも非対応だったが、作者がIssue立てて対応中。

https://github.com/onikan27/claude-code-monitor/issues/7

環境変数ベースでシンプルに実装される予定：

```bash
# WezTermは子プロセスに環境変数を継承
$ echo $WEZTERM_PANE
13

# これで直接フォーカス移動
$ wezterm cli activate-pane --pane-id $WEZTERM_PANE
```

## まとめ

- **claude-code-monitor**: 複数セッション監視に必須
- **Zedの内蔵ターミナル**: ccmのフォーカス切替が使えない
- **WezTerm + LazyVim**: ccmをフル活用できる構成に移行

Claude Codeを複数並行で動かすなら、ccmは入れておいて損はない。

## 参考リンク

- [claude-code-monitor](https://github.com/onikan27/claude-code-monitor)
- [WezTerm](https://wezfurlong.org/wezterm/)
- [LazyVim](https://www.lazyvim.org/)
