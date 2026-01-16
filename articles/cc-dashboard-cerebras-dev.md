---
title: "Cerebras × GLM-4.7でClaude Code用ステータスラインを爆速開発した"
emoji: "⚡"
type: "tech"
topics: ["claudecode", "cerebras", "go", "cli"]
published: false
---

## はじめに

> 業務自動化Pythonエンジニア。バイブコーディング歴1年 ≒ エンジニア歴。

### きっかけ

API課金したので、会社のプロジェクトではなく個人の作業をしたかった。

ずっと欲しかったのが**4分割でも綺麗に見えるstatusline**。

既存のccstatuslineやccusageを試したけど、weeklyのデータやsessionの取得が上手くいかなかった記憶がある。

ccstatusline + ccusageをめっちゃ参考にしつつ、そのままコードをパクるだけだと面白くないので**Goで書いてみた**。

### Cerebrasとは

以前の記事で紹介したCerebras × GLM-4.7。1,000トークン/秒の爆速推論。

「これでコーディングしたら速いのでは？」と思って、実際に作ってみた。

## 作ったもの：cc-quadstat

Claude Code用のPowerlineスタイルのステータスライン。

```
 Opus 4.5  v2.1.6  ⎇ main (+42,-10)
 🧠 78%  ⏱ 5h: 3% (4h17m)
 📅 All: 0% (火20:00)  💬 Sonnet: 0%
```

| 行 | 内容 |
|----|------|
| 1行目 | モデル名、Claude Codeバージョン、ブランチ名、変更数 |
| 2行目 | コンテキスト残量、5時間使用量（リセット時間） |
| 3行目 | 週間使用量（全モデル）、Sonnet使用量 |

### なぜGoで書いたか

既存ツールはTypeScript製。

正直なところ、同じ言語だと「参考にした」感が出すぎるので別の言語にした。

あとGoは単一バイナリになるのでインストールが楽。

### 4分割でも見やすく

ターミナルを4分割して使うと、1行が長いと折りたたまれたり省略されたりする。

だから**1行を短く設計**した。

- 1行目：セッション情報（モデル、バージョン、VCS）
- 2行目：今のセッションの状態（コンテキスト、5時間制限）
- 3行目：週間の残量

自分が本当に必要な情報だけを詰め込んだ。

## 機能

### 8種類のテーマ

```bash
CC_THEME=dracula ~/.claude/scripts/cc-quadstat
```

| テーマ | 説明 |
|--------|------|
| tokyo-night | デフォルト。青/紫系 |
| dracula | 紫と緑、高コントラスト |
| nord | 北欧風、青系 |
| gruvbox | レトロ、暖色系 |

### 色分け

**コンテキスト残量（🧠）**
- 50%以上：緑
- 20-50%：黄
- 20%未満：赤

**使用量（⏱📅💬）**
- 50%未満：緑
- 50-80%：黄
- 80%以上：赤

### VCS対応

GitとJujutsu（jj）の両方に対応。

```
⎇ main (+42,-10)
```

変更数もわかる。

## インストール

### リリースから（推奨）

```bash
# macOS (Apple Silicon)
curl -L https://github.com/kok1eee/cc-quadstat/releases/latest/download/cc-quadstat_darwin_arm64.tar.gz | tar xz

# 配置
mkdir -p ~/.claude/scripts
mv cc-quadstat ~/.claude/scripts/
chmod +x ~/.claude/scripts/cc-quadstat
```

### 設定

初期設定コマンドで自動設定：

```bash
cc-quadstat --init
```

または手動で `~/.claude/settings.json` に追加：

```json
{
  "statusLine": {
    "type": "command",
    "command": "~/.claude/scripts/cc-quadstat"
  }
}
```

### CLIコマンド

```bash
# テーマ一覧を表示
cc-quadstat --list-themes

# テーマを変更
cc-quadstat --set-theme dracula

# ヘルプ
cc-quadstat --help
```

テーマ設定は `~/.config/cc-quadstat/theme` に保存される。環境変数 `CC_THEME` でも上書き可能。

## Cerebrasでの開発体験

### 良かった点

- **速い**：考える速度で返ってくる
- **日本語対応**：GLM-4.7は日本語が得意

### 課題

- **ハーネスの違い**：OpenCode経由で使ったが、Claude Codeほど使い慣れていない
- **rate limit**：課金してもレートエラーが発生

完成まで**7ドル**かかった。結局、途中からClaude Codeに戻って仕上げた。

やっぱり自分はAPI課金だと「今いくらかかってるんだろう」が気になってしまう。サブスクの方が精神的に安心して使える。

## 既知の問題：built-in表示との競合

Claude Code v2.1.x系のどこかで、組み込みのgit diff表示機能が追加されたようだ（changelogには記載なし）。

これにより、cc-quadstatの出力の下にClaude Code側のgit diff情報が重複表示されるようになった。

```
 Opus 4.5  v2.1.9  ⎇ r (+9930,-7)      ← cc-quadstat（変更数表示）
 🧠 72%  ⏱ 5h: 8% (2h44m)
 📅 All: 18% (火19:59)  💬 Sonnet: 1%
  21 files +9930 -7                      ← Claude Code built-in（重複）
```

現時点ではこのbuilt-in表示を無効にする設定オプションは存在しない。

Feature Requestを作成済み：[#18475](https://github.com/anthropics/claude-code/issues/18475)

## まとめ

- Cerebras × GLM-4.7で爆速開発を体験
- 自分専用の最強ステータスラインを作成
- Goで書いてインストールを簡単に

GitHub: [kok1eee/cc-quadstat](https://github.com/kok1eee/cc-quadstat)

## 参考

- [ccusage](https://github.com/ryoppippi/ccusage) - 参考にしたツール
- [ccstatusline](https://github.com/ryoppippi/ccstatusline) - 参考にしたツール
