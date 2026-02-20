---
title: "Claude Code 2.1.49 の新機能を自作プラグインに入れてみた"
emoji: "🧪"
type: "tech"
topics: ["claudecode", "ai", "plugin", "multiagent"]
published: true
---

## はじめに

> 業務自動化Pythonエンジニア。バイブコーディング歴1年 ≒ エンジニア歴。

Claude Code 2.1.49 のリリースノートを読んでいたら、自作プラグイン [o-m-cc](https://github.com/kok1eee/o-m-cc) に関係しそうな機能が3つあった。

- プラグインに `settings.json` を同梱できるようになった
- エージェント定義に `background: true` を追加できるようになった
- エージェント定義に `isolation: "worktree"` を追加できるようになった

どれも公式ドキュメントにはまだ書かれていない。GitHub Issues で「ドキュメント不足」として報告されている状態。せっかくなので実験的に全部入れてみた。

前回の記事:

https://zenn.dev/kok1eee/articles/o-m-cc-takt-inspired-update

## 3つの新機能

### 1. プラグインに settings.json を同梱

**何ができるか**: プラグインのインストール時にデフォルト設定を自動で提供できる。

今までは `/install` や `/init` スキルで手動セットアップしていた設定を、プラグインに同梱できる。

```json
// .claude-plugin/settings.json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  },
  "agent": "sisyphus",
  "spinnerVerbs": {
    "mode": "replace",
    "verbs": [
      "岩を押し上げています",
      "山頂を目指しています",
      "また麓から登っています"
    ]
  },
  "permissions": {
    "allow": [
      "Read(plan/**)",
      "Write(plan/**)",
      "Edit(plan/**)",
      "Glob(**)",
      "Grep(**)"
    ]
  }
}
```

**o-m-cc での活用**:

| 設定 | 今まで | settings.json で |
|------|--------|----------------|
| spinnerVerbs | `/install` で手動追加 | インストール時に自動 |
| permissions | `/init` で手動マージ | インストール時に自動 |
| Agent Teams 有効化 | ユーザーが手動設定 | インストール時に自動 |
| デフォルトエージェント | ユーザーが手動設定 | インストール時に自動 |

セットアップの摩擦がかなり減る。ただし、ユーザー設定との**マージの優先順位がドキュメント化されていない**ので、上書きされるのか未設定時のみ埋まるのかは不明。

### 2. エージェントの `background: true`

**何ができるか**: 特定のエージェントを常にバックグラウンドで実行させるヒント。

```yaml
---
name: researcher
description: 公式ドキュメント、実装例の調査
tools: Read, Glob, Grep, WebSearch, WebFetch
model: sonnet
background: true
---
```

I/O集約的な調査エージェントは結果を待つ必要がないことが多い。`background: true` を付けると、Claude Code が自動的にバックグラウンドで実行してくれる。

**o-m-cc で付けたエージェント**:

| エージェント | 理由 |
|-------------|------|
| researcher | Web検索・ドキュメント調査はI/O待ち |
| learnings-researcher | VCS履歴の検索はI/O待ち |
| explore | コードベース探索は読み取り専用 |

**実際のテスト結果**: `run_in_background` パラメータを明示せずに explore を呼び出したところ、**自動的にバックグラウンドで実行された**。機能は動作確認済み。

### 3. エージェントの `isolation: "worktree"`

**何ができるか**: エージェントが一時的な git worktree で実行される。メインの作業ツリーに影響を与えずにファイル変更ができる。

```yaml
---
name: frontend
description: フロントエンド実装、UI設計、コンポーネント作成
tools: Read, Write, Edit, Glob, Grep
model: sonnet
isolation: worktree
---
```

複数のエージェントが同時にファイルを変更する場合、worktree で分離すれば競合しない。

**o-m-cc で付けたエージェント**:

| エージェント | 理由 |
|-------------|------|
| frontend | UIコンポーネントの作成・編集 |
| designer | 設計書（design.md）の作成 |
| planner | タスク定義（tasks.md）の作成 |
| debugger | バグ修正でコード変更 |

## 公式ドキュメントの状況

3つとも 2.1.49 のチェンジログには記載があるが、ドキュメントは未更新。GitHub Issues で報告されている:

| 機能 | Issue |
|------|-------|
| settings.json | [#27026](https://github.com/anthropics/claude-code/issues/27026) |
| background: true | [#27025](https://github.com/anthropics/claude-code/issues/27025) |
| isolation: worktree | [#27023](https://github.com/anthropics/claude-code/issues/27023) |

サブエージェントの[公式ドキュメント](https://code.claude.com/docs/en/sub-agents)に記載されているフロントマッターフィールドは `name`, `description`, `tools`, `disallowedTools`, `model`, `permissionMode`, `maxTurns`, `skills`, `mcpServers`, `hooks`, `memory` の11個。`background` と `isolation` はまだ載っていない。

公式プラグイン（code-review, feature-dev, security-guidance）もまだどれも使っていない。

## やってみた感想

### 良かった点

- **未知のフロントマッターフィールドはエラーにならない**。安全に追加できる
- **`background: true` は確実に動作する**。テストで確認済み
- **settings.json はキャッシュにコピーされる**。プラグインの一部として認識されている

### 注意点

- **`plugin update` はバージョン番号が変わらないとキャッシュを更新しない**。開発中は `uninstall` → `install` が必要
- **settings.json のマージ挙動はドキュメント化されていない**。ユーザー設定との優先順位が不明
- **isolation: worktree の動作は未確認**。worktree が実際に作成されたかは出力からは判別できなかった

### バックアップは必須

実験的な機能を入れるので、変更前に jj のブックマークでバックアップを取った。

```bash
jj bookmark create backup-v0.15.1 -r main
```

問題があれば `jj rebase -r @ -d backup-v0.15.1` で即座に戻せる。

## まとめ

Claude Code 2.1.49 の新機能をプラグインに入れてみた結果:

| 機能 | 動作 | 実用度 |
|------|:----:|:------:|
| settings.json | キャッシュ反映済み | ドキュメント待ち |
| background: true | **動作確認済み** | すぐ使える |
| isolation: worktree | エラーなし | 要実プロジェクトテスト |

ドキュメントが整備されるのを待ってもいいが、`background: true` のように既に動くものもある。バックアップを取りつつ実験するのが一番学びが多い。

リポジトリ: https://github.com/kok1eee/o-m-cc
