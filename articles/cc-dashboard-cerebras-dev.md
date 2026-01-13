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

Claude Codeのステータスラインをカスタマイズしたかった。

既存のツール（ccusage、ccstatusline）を参考にしつつ、自分専用の最強パーツを作りたくなった。

### Cerebrasで作った理由

以前の記事でCerebras × GLM-4.7を試した。1,000トークン/秒の爆速推論。

「これでコーディングしたら速いのでは？」と思って、実際に作ってみた。

## 作ったもの：cc-dashboard

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

正直なところ、**バレにくくするため**にGoにした。同じ言語だと「参考にした」感が出すぎる。

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
CC_THEME=dracula ~/.claude/scripts/cc-dashboard
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
curl -L https://github.com/tazawa-masayoshi/cc-dashboard/releases/latest/download/cc-dashboard_darwin_arm64.tar.gz | tar xz

# 配置
mkdir -p ~/.claude/scripts
mv cc-dashboard ~/.claude/scripts/
chmod +x ~/.claude/scripts/cc-dashboard
```

### 設定

`~/.claude/settings.json` に追加：

```json
{
  "statusLine": {
    "type": "command",
    "command": "CC_THEME=dracula ~/.claude/scripts/cc-dashboard"
  }
}
```

## Cerebrasでの開発体験

### 良かった点

- **速い**：考える速度で返ってくる
- **日本語対応**：GLM-4.7は日本語が得意

### 課題

- **ハーネスの違い**：OpenCode経由で使ったが、Claude Codeほど使い慣れていない
- **rate limit**：Freeプランだと毎回1分待ち

結局、途中からClaude Codeに戻って仕上げた。

## まとめ

- Cerebras × GLM-4.7で爆速開発を体験
- 自分専用の最強ステータスラインを作成
- Goで書いてインストールを簡単に

GitHub: [tazawa-masayoshi/cc-dashboard](https://github.com/tazawa-masayoshi/cc-dashboard)

## 参考

- [ccusage](https://github.com/ryoppippi/ccusage) - 参考にしたツール
- [ccstatusline](https://github.com/ryoppippi/ccstatusline) - 参考にしたツール
