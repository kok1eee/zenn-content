---
title: "Claude Codeの日本語問題を機にOpenCode + oh-my-opencodeを試してみた"
emoji: "🚀"
type: "tech"
topics: ["opencode", "ai", "cli", "claude", "gemini"]
published: true
---

## はじめに

> 業務自動化Pythonエンジニア。バイブコーディング歴1年 ≒ エンジニア歴。

### Claude Code大好きなんだけど

毎日チェンジログを見るくらいClaude Codeが好き。新機能追加されるたびにワクワクする。

でも最近、日本語環境で問題が発生してる。

### 日本語だとRust panicする問題

最新バージョンで日本語を使うと、こんなエラーが頻発する：

```
Rust panic: byte index not char boundary with Japanese text
```

**日本語ユーザーにはなかなかキツい。**

:::message
**2025/1/8 追記**: Claude Code v2.1.0で「CJK改行問題修正」がリリースされました。日本語の改行表示バグが修正されたかもしれません。
:::

治るまでの間、代替ツールを探してた。そしたら見つけてしまった。

### oh-my-opencodeが面白そう

「複数のAIモデルを協調させる」って何それ？

Claude、GPT、Geminiを目的別に使い分ける？エージェントが連携する？

あと、メインエージェントの名前が **Sisyphus**（シシュポス）。かっこいい。

**以前 Super Claude を見てワクワクした人は、きっと刺さる。**

Super Claudeは専門家エージェントが多すぎて「使いこなせないな...」と感じた人もいると思う。oh-my-opencodeはその辺りがちょうどいい。必要なものだけ入ってる感じ。

というわけで、導入してみた。

## OpenCode とは

[OpenCode](https://github.com/sst/opencode) は、SST が開発するオープンソースのAIコーディングエージェント。

**特徴:**
- 無限にカスタマイズ可能
- 画面のちらつきがない
- LSP、リンター、フォーマッターが自動で有効化
- 複数モデルを目的別に使い分け可能

## oh-my-opencode とは

[oh-my-opencode](https://github.com/code-yeongyu/oh-my-opencode) は、OpenCode を強化するプラグイン。

**同梱されているもの:**
- **エージェント**: Sisyphus, Oracle, Librarian, Explore など
- **MCP**: Context7, Exa (Web検索), grep_app (GitHub検索)
- **ツール**: LSP/AST サポート、Todo継続、コメントチェッカー
- **Claude Code 互換レイヤー**

## インストール

### 1. OpenCode のインストール

mise を使う場合:

```bash
# GitHub attestations を無効化（レート制限回避）
mise settings github.github_attestations false

# インストール
mise use --pin -g "github:sst/opencode@1.1.1"
```

### 2. oh-my-opencode のインストール

```bash
npx oh-my-opencode install --no-tui --claude=yes --chatgpt=no --gemini=no
```

オプション:
- `--claude=yes|no|max20`: Claude Pro/Max サブスク
- `--chatgpt=yes|no`: ChatGPT Plus/Pro サブスク
- `--gemini=yes|no`: Google AI Pro サブスク

### 3. 認証

```bash
opencode auth login
# Provider: Anthropic を選択
# Login method: Claude Pro/Max を選択
# ブラウザで認証
```

## エージェント構成

oh-my-opencode には複数のエージェントが含まれている：

| エージェント | 本来のモデル | 役割 |
|-------------|-------------|------|
| **Sisyphus** | Claude Opus 4.5 | メインオーケストレーター |
| **Oracle** | GPT 5.2 | 設計・デバッグ |
| **Frontend** | Gemini 3 Pro | UI/UX 開発 |
| **Librarian** | Claude Sonnet 4.5 | ドキュメント・実装例検索 |
| **Explore** | Grok Code | 高速コード探索 |

### Sisyphusの元ネタ？：Claude CodeのRalph Wiggum

oh-my-opencodeの「Sisyphus」や「Ralph Loop」は、**Claude Code公式プラグインの「Ralph Wiggum」にインスパイアされているっぽい**。

Ralph Wiggumはタスクが完了するまで実行し続ける自己参照型開発ループ。oh-my-opencodeではこれを`ralph-loop`として実装している：

- `/ralph-loop "REST API を構築"` で開始
- `<promise>DONE</promise>` の出力で完了を検知
- 完了プロミスなしで停止すると自動再開
- 終了条件: 完了検知、最大反復回数到達（デフォルト100）、または `/cancel-ralph`

岩を転がし続けるSisyphusの大元は、Claude Codeにもいたラルフくんだった。

### 私の場合：無料モデルで運用

ChatGPT や Gemini のサブスクは持ってない。Claude Pro だけ。

でも問題なく動く。oh-my-opencode の設定で、各エージェントのモデルを変更できる。私は無料モデルや Claude に振り替えて使ってる。

## 同梱 MCP

何も設定しなくても以下が使える：

| MCP | 用途 |
|-----|------|
| **context7** | ライブラリのドキュメント検索 |
| **websearch_exa** | Web 検索 |
| **grep_app** | GitHub コード検索 |

### 追加 MCP（オプション）

| MCP | 用途 | いつ必要？ |
|-----|------|-----------|
| Exa (自前APIキー) | レート制限回避 | ヘビーユーザー |
| Morph | 高速コードベース検索 | 大規模プロジェクト |

## 使い方

```bash
opencode
# または alias を設定
alias oc="opencode"
```

プロンプトに `ultrawork` を含めると、全機能がフル稼働する。

## OpenCode の良いところ

使ってみて良かった点をまとめる。

### サイドバーで状態が見える

Claude Code のステータスラインに相当する機能。画面右側のサイドバーに情報が表示される。

**表示される情報：**
- **Context**: コンテキスト使用量
- **MCPs**: 接続中のMCPサーバー
- **LSP**: 言語サーバーの状態
- **Modified Files**: 変更されたファイル一覧

作業中の状態が一目でわかるのは地味に助かる。

現時点ではサイドバーの表示内容は固定。でも [Issue #5971](https://github.com/sst/opencode/issues/5971) で機能拡張が議論されてる。今後に期待。

### compactの中身が見れる

コンテキストを圧縮したときに、中身が確認できる。何が残ってるかわかるので安心感がある。

実際のcompact後の表示例：

```
継続プロンプト
プロジェクト概要
〇〇プロジェクトの△△機能をリファクタリング中。
パフォーマンス最適化とConfig駆動化を実施。
---
完了した作業
1. Config駆動化
2. パフォーマンス最適化
---
次にやること
1. 動作確認
2. 後回しタスク（TODO登録済み）
```

何を覚えているかが明確にわかる。

### todo継続hookが働く

タスクの途中で別の作業を依頼しても、未完了タスクを続けさせようとしてくれる。

VIBERで横道に逸れがちな自分には助かる。Sisyphusの名前通り、ひたすら石を押し続けてくれる感じ。

### リスク評価でタスクをキャンセルしてくれる

ただ石を押し続けるだけじゃない。タスクのリスクを評価して、危険な変更はキャンセルしてくれる。

```
リスク評価:
- 本番稼働中の機能に影響
- 低優先度タスクとしては大きすぎる変更

決定: 現在の状態で削除すると破壊的変更になる。
TODO項目をキャンセルし、理由を記録します。
```

勝手に突き進まないのは信頼できる。

### メッセージ操作が直感的

Claude Code だと `Esc` 2回で Rewind メニューが出る。

OpenCode はメッセージを直接クリックして、3つの選択肢が出る：
- **revert**: メッセージとファイル変更を戻す
- **copy**: メッセージをクリップボードにコピー
- **fork**: 新しいセッションを作成

直感的でわかりやすい。

### ターミナル選択で自動コピー

ターミナルの出力を選択すると、勝手にクリップボードにコピーされる。どうせコピーするし、地味に便利。

## Claude Code との違い（デメリット）

良いところばかりじゃない。Claude Code の方が楽な点もある。

### セッション再開がちょっと手間

Claude Code だと `--continue` でカレントディレクトリの会話を再開できる。

OpenCode の場合：
- `--continue`: グローバルで最後のセッションを再開
- 特定のセッションを再開したい場合は、起動後に `Ctrl+P` → `Switch session` で探す

ディレクトリごとの自動再開はない。ここは Claude Code の方が楽。

### 画像を直接貼れない

Claude Code だとターミナルに画像をドラッグ&ドロップできる。OpenCode は今のところ対応してない。

### 2分割くらいがちょうどいい

ターミナルを4分割して使うと、サイドバーが右側ではなく上部に移動する。画面が狭いと判断されるみたい。正直ちょっと邪魔。

アウトプットの中身をある程度見たい派なので、2分割くらいが体験良さそう。小さすぎると見づらい。

### Claude Code OAuthは使えなくなった

:::message alert
**2025/1/9 追記**: Claude Code の OAuth トークンを OpenCode で使うことはできなくなりました。
:::

以前は特殊なヘッダーで動作していたらしいが、Anthropic が対策した模様。

```
This credential is only authorized for use with Claude Code and cannot be used for other API requests.
```

**OpenCode で Claude を使いたい場合は、別途 API キーを発行する必要がある。**

- Claude Max/Pro サブスクリプション → Claude Code 専用
- API キー（従量課金） → OpenCode 等の他ツール用

参考: [sst/opencode#417](https://github.com/sst/opencode/issues/417)

:::details 回避策（友人情報）

[PR #10](https://github.com/anomalyco/opencode-anthropic-auth/pull/10) で修正が進んでいる。友人から簡単な適用方法を教えてもらった。

1. [anomalyco/opencode](https://github.com/anomalyco/opencode) をClone
2. `packages/opencode/src/plugin/index.ts` の14行目あたりを編集：
   ```diff
   - const BUILTIN = ["opencode-copilot-auth@0.0.9", "opencode-anthropic-auth@0.0.5"]
   + const BUILTIN = ["opencode-copilot-auth@0.0.9", "opencode-anthropic-auth@0.0.6"]
   ```
3. ビルド：`cd opencode/packages/opencode && bun run build -- --single`
4. PATHを通す：
   ```bash
   export PATH=/path/to/opencode/packages/opencode/dist/opencode-darwin-arm64/bin/:$PATH
   ```

正式リリースで同じ修正が来るはずなので、先回りしたい人向け。

:::

## 設定ファイル

### ~/.config/opencode/opencode.json

```json
{
  "$schema": "https://opencode.ai/config.json",
  "plugin": [
    "oh-my-opencode"
  ]
}
```

### ~/.config/opencode/oh-my-opencode.json

エージェントのモデルをカスタマイズできる：

```json
{
  "agents": {
    "librarian": { "model": "opencode/free-model" },
    "explore": { "model": "opencode/free-model" },
    "oracle": { "model": "anthropic/claude-opus-4-5" }
  }
}
```

## 参考リンク

- [OpenCode 公式](https://opencode.ai/)
- [oh-my-opencode GitHub](https://github.com/code-yeongyu/oh-my-opencode)
- [設定例リポジトリ](https://github.com/safzanpirani/opencode-configs)

## まとめ

OpenCode + oh-my-opencode で、複数のAIモデルを協調させるコーディング環境が作れる。まずは Claude Pro だけで始めて、物足りなければ ChatGPT や Gemini を追加していけばいい。

まだ使い始めなので、発見があればガンガン追記していく予定。
