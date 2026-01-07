---
title: "海外で「IDEでClaude Code動かすな」が流行ってる。時代が俺に追いついたか"
emoji: "💻"
type: "tech"
topics: ["claudecode", "zed", "opencode", "terminal", "wezterm"]
published: true
---

## はじめに

> 業務自動化Pythonエンジニア。バイブコーディング歴1年 ≒ エンジニア歴。

### Xでこんなのが流れてきた

海外で「Claude CodeをCursorやVS Codeで動かすのをやめろ」という主張が話題になってる。

要約するとこんな感じ：

- **ターミナルで直接実行しろ**。メモリ効率が段違い
- PCが速くなるし、**複数ターミナル同時起動できる**
- IDEはマジで重い。**Ghostty（無料）** なら画面分割して片方サーバー、片方Claude
- IDE卒業した。**Opus 4.5が賢すぎてもうコード見なくていい**
- 環境変数変えたいときだけFinderからテキストエディタで開けばOK

### なるほど、時代が俺に追いついてきたか

正直、「コード見なくていい」は言い過ぎだと思う。

でも、**IDE重い問題は前から感じてた**。

だから俺はずっと**Zed**を使ってる。

## Zedのターミナルがいい

### 軽い

まずこれ。Rust製なのでメモリ効率が段違い。

| エディタ | メモリ使用量 | 備考 |
|---------|-------------|------|
| Cursor | 約1GB〜 | Electron製。AI機能が重い |
| VS Code | 約800MB〜 | Electron製 |
| **Zed** | 約200MB | Rust製 |

Cursorの1/5。体感でも全然違う。

### ボタン一つでターミナル分割

Ctrl+aとかLeaderキーとか覚えなくていい。

ターミナルパネルのボタンをポチポチするだけで分割できる。直感的。

### 4分割できる

**VS Codeフォークは横並びしかできない**。

これがマジで嫌で、Cursor、Windsurf、Antigravity...全部アレルギーが出てしまう。

Zedは**tmuxのように縦横自由に分割**できる。4分割も余裕。

### タブも作れる

しかもターミナルにタブを作れる。

つまり、**4分割 × 複数タブ**で、いくつものプロジェクトを一気に並列で回せる。

### コード隠せる

`Ctrl+J`でターミナルパネルを最大化/非表示できる。

基本的にターミナル全画面で作業して、コードが見たくなったら`Ctrl+J`で戻る。

Claude Codeに集中できる。

## 俺のワークフロー

### メイン：Zed、サブ：WezTerm

Zedだけで完結することもあるけど、並列作業が増えてきたらWezTermも併用する。

最近はOpenCodeも使ってるので：

- **WezTerm**: OpenCode 2分割
- **Zed**: Claude Code 4分割

**俺は6分割までで十分・・・！（つーか これが限界）**

### Claude CodeとOpenCodeの使い分け

セッション管理の仕方が違うから、使い分けてる。

**Claude Code**：ディレクトリに行って`cc -c`

```bash
cd ~/projects/my-app
cc -c  # 直前のセッション再開
```

これが気持ちいい。ちょっとした修正はこれでサクッと。

**OpenCode**：`/session`で探す

起動してから`/session`コマンドでセッション一覧を出す。ディレクトリと紐づいてない。

長い作業はOpenCodeの方がセッション管理しやすい場面もある。

## 環境構成

参考までに、自分の構成を晒しておく。

| カテゴリ | ツール |
|---------|--------|
| エディタ | **Zed** |
| ターミナル | **WezTerm** |
| シェル | zsh + **sheldon** |
| プロンプト | **powerlevel10k** |

**Rust製で固めてる**。全部軽い。

### Zedの設定

```json
{
  "vim_mode": true,
  "base_keymap": "Cursor",
  "theme": {
    "dark": "Dracula"
  },
  "terminal": {
    "copy_on_select": true
  }
}
```

Cursorのキーマップベースにしつつ、Vimモード。

**Claude Codeが設定ファイルを理解してカスタマイズしてくれる**のも地味に便利。

### WezTermの設定

tmux風にカスタマイズしてる。

```lua
-- tmux風のLeaderキー
config.leader = { key = "a", mods = "CTRL", timeout_milliseconds = 2000 }

-- 背景透過 + ぼかし
config.window_background_opacity = 0.85
config.macos_window_background_blur = 20
```

| キー | 動作 |
|-----|------|
| `Ctrl+a -` | 縦分割 |
| `Ctrl+a \` | 横分割 |
| `Ctrl+a 1〜4` | ペイン移動 |
| `Ctrl+a z` | ペイン最大化 |

WezTermもLuaなので**Claude Codeが設定いじってくれる**。

### sheldon（プラグインマネージャー）

Rust製のzshプラグインマネージャー。起動が速い。

```toml
[plugins.powerlevel10k]
github = "romkatv/powerlevel10k"

[plugins.zsh-autosuggestions]
github = "zsh-users/zsh-autosuggestions"

[plugins.zsh-abbr]
github = "olets/zsh-abbr"

[plugins.zsh-syntax-highlighting]
github = "zsh-users/zsh-syntax-highlighting"
```

`zsh-autosuggestions`で過去のClaude Codeコマンドが補完されるのが便利。

## 完全ターミナル派への反論

海外勢の「IDE卒業」「コード見なくていい」は極論だと思う。

- **差分確認**したいときがある
- **コードレビュー**で構造を把握したい
- **学習目的**で実装を読みたい

Opus 4.5が賢いのは事実。でも全部CLIは無理がある。

**軽いエディタを使えばいい**。それがZed。

## まとめ

| 選択肢 | メモリ | おすすめ |
|-------|-------|---------|
| Cursor/VS Code | 重い | 今のままでいい人 |
| **Zed** | 軽い | ターミナル分割したい人 |
| WezTerm単体 | 軽い | ガチターミナル派 |
| Ghostty単体 | 最軽量 | ミニマリスト |

海外で「IDEやめろ」が流行ってるけど、**Zedなら軽さとエディタ機能を両立できる**。

VS Codeフォークの横並びしかできないターミナルにアレルギーある人、Zed試してみて。

---

**みんなはどうしてる？** Claude Code、どこから叩いてる？

## あとがき：Rust製だったら飛びつくPython使い

uvに感動して、Polarsを触ってみて、miseやZedに感動を覚えっぱなし。

気づいたらターミナルもシェルもRust製。

先輩エンジニアには「新しいものより安定を求めるのが一番」とよく言われる。

でも、それを聞くたびに思う。**今の自分は禅だな**と。

全てのものをワクワクしながら取り入れる。毎日Claude Codeと禅問答。知らない技術を言われたら取り入れて、よかったら過去のコードも都度リファクタしちゃう。

AI時代、全てのものをキャッチアップするのは難しい。

でも、楽しみながら目に見えた範囲から**やっていき**。
