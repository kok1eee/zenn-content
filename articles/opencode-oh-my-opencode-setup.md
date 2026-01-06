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

oh-my-opencode には複数のエージェントが含まれています:

| エージェント | 本来のモデル | 役割 |
|-------------|-------------|------|
| **Sisyphus** | Claude Opus 4.5 | メインオーケストレーター |
| **Oracle** | GPT 5.2 | 設計・デバッグ |
| **Frontend** | Gemini 3 Pro | UI/UX 開発 |
| **Librarian** | Claude Sonnet 4.5 | ドキュメント・実装例検索 |
| **Explore** | Grok Code | 高速コード探索 |

### 私の場合：無料モデルで運用

ChatGPT や Gemini のサブスクは持ってない。Claude Pro だけ。

でも問題なく動く。oh-my-opencode の設定で、各エージェントのモデルを変更できる。私は無料モデルや Claude に振り替えて使ってる。

## 同梱 MCP

何も設定しなくても以下が使えます:

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

## サイドバーが便利

Claude Code のステータスラインに相当する機能がある。画面右側のサイドバーに情報が表示される。

**表示される情報：**
- **Context**: コンテキスト使用量
- **MCPs**: 接続中のMCPサーバー
- **LSP**: 言語サーバーの状態
- **Modified Files**: 変更されたファイル一覧

作業中の状態が一目でわかるのは地味に助かる。

### 将来的にはカスタマイズできそう

現時点ではサイドバーの表示内容は固定。でも [Issue #5971](https://github.com/sst/opencode/issues/5971) で機能拡張が議論されてる。

oh-my-opencode の作者も、バックグラウンドエージェントのステータス表示とか欲しがってるみたい。今後に期待。

### 2分割くらいがちょうどいい

ターミナルを4分割して使うと、サイドバーが右側ではなく上部に移動する。画面が狭いと判断されるみたい。正直ちょっと邪魔。

アウトプットの中身をある程度見たい派なので、2分割くらいが体験良さそう。小さすぎると見づらい。

## 使い方

```bash
opencode
# または alias を設定
alias oc="opencode"
```

プロンプトに `ultrawork` を含めると、全機能がフル稼働する。

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
