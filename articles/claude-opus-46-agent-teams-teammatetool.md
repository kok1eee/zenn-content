---
title: "Opus 4.6 の Agent Teams でマルチエージェントオーケストレーターを強化する"
emoji: "🤝"
type: "tech"
topics: ["claudecode", "ai", "cli", "multiagent", "opus"]
published: true
---

:::message
**TL;DR** — Opus 4.6 の Agent Teams（peer-to-peer 協調）で、o-m-cc のマルチエージェントを subagent から全面移行した。エージェント同士が議論して結論を出せるようになった。
:::

## はじめに

> 業務自動化Pythonエンジニア。バイブコーディング歴1年 ≒ エンジニア歴。

2026年2月5日、Anthropic が Claude Opus 4.6 をリリースした。目玉機能の一つが **Agent Teams**。複数の Claude Code セッションが peer-to-peer で協調作業できる。

以前の記事で紹介した自作プラグイン [o-m-cc](https://github.com/kok1eee/o-m-cc) は、Task tool (subagent) ベースのマルチエージェントオーケストレーターだった。今回 Agent Teams に全面移行して v0.8.0 にした。subagent では「結果を返すだけ」だったエージェントが、互いに議論して結論を出せるようになった。

この記事では Agent Teams の仕組みと、既存のマルチエージェントワークフローをどう強化できるかを共有する。自分でオーケストレーターを組んでいる人の参考になれば。

https://zenn.dev/kok1eee/articles/o-m-cc-claude-code-plugin

## Agent Teams とは

### 従来: Task tool (subagent)

```
メインエージェント
  ├─ Task(explore): 「src/auth/ にあります」 → 結果を返して終了
  ├─ Task(code-reviewer): 「問題3件」 → 結果を返して終了
  └─ メインが全てを統合
```

subagent は「結果を返すだけの使い捨てワーカー」。互いの存在を知らない。

### Agent Teams (TeammateTool)

```
Lead（メインエージェント）
  ├─ spawnTeam: チーム作成
  ├─ spawnTeammate: code-reviewer → 「XSS の可能性あり」
  ├─ spawnTeammate: security-reviewer → 「それ確認した、CSP で緩和可能」
  │       ↕ peer-to-peer メッセージ交換
  └─ Lead が統合レポート作成
```

teammates は**互いにメッセージを送り合える**。議論して、より良い結論に辿り着く。

### 比較

|                   | Subagent | Agent Teams |
| :---------------- | :------- | :---------- |
| **通信** | メインにだけ返す | teammate 同士でメッセージ交換 |
| **調整** | メインが全管理 | 共有タスクリスト + 自己調整 |
| **向いている** | 結果だけ欲しい単発タスク | 議論・協調が必要な複雑な作業 |
| **トークンコスト** | 低い | 高い（各 teammate が独立セッション） |

> 参考: [Orchestrate teams of Claude Code sessions](https://code.claude.com/docs/en/agent-teams)

## 有効化

Agent Teams はデフォルト無効。

```json
// ~/.claude/settings.json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

## o-m-cc を Agent Teams で強化する

### o-m-cc のおさらい

[o-m-cc](https://github.com/kok1eee/o-m-cc) は Claude Code 用のマルチエージェントプラグイン。16の専門エージェント、Sisyphus Loop（タスク完了まで止まらない）、仕様駆動開発フロー、hooks による自動化を提供する。

v0.7 までは Task tool (subagent) でエージェントを起動していた。subagent は結果を返すだけで、互いの存在を知らない。これを Agent Teams に置き換えることで、エージェント同士が議論・協調できるようになった。

### 何が変わったか

| 項目 | v0.7 (Task tool) | v0.8 (Agent Teams) |
|------|-----------------|-------------------|
| エージェント起動 | `Task tool で subagent を呼び出し` | `spawnTeam → teammate を spawn` |
| 通信 | 親→子の一方向 | peer-to-peer メッセージング |
| 並列実行 | `background=true` で並列 | teammates が自律的に並列作業 |
| タスク管理 | メインが全管理 | 共有タスクリスト + 自己調整 |
| コマンドの allowed-tools | `Task` | `TeammateTool` |

## 強化パターン: subagent → Agent Teams

ここから実践。o-m-cc で実際に移行したパターンを紹介する。

### パターン1: 相互レビュー（/review）

最もシンプルな Agent Teams 活用。2つの専門エージェントが互いの発見を共有する。

**Before (subagent)**:
```
Task tool で並列実行（両方 background=true）：
1. code-reviewer subagent → 結果を返す
2. security-reviewer subagent → 結果を返す
→ メインが統合
```

**After (Agent Teams)**:
```markdown
## Step 1: チーム作成
spawnTeam: "review"

## Step 2: Teammates Spawn
spawnTeammate: code-reviewer
  prompt: |
    agents/code-reviewer.md の指示に従いレビュー。
    発見した問題を security-reviewer teammate にもメッセージで共有。

spawnTeammate: security-reviewer
  prompt: |
    agents/security-reviewer.md の指示に従いチェック。
    発見した問題を code-reviewer teammate にもメッセージで共有。

## Step 3: 議論・集約
→ teammates が互いの発見をメッセージで共有
→ Lead が両者の結果を統合レポートに
```

**ポイント**: spawn prompt に「他の teammate にメッセージで共有」と明記する。これがないと subagent と変わらない動きになる。

### パターン2: 並列 + 順次ハイブリッド（/plan）

独立したフェーズは並列、依存関係があるフェーズは順次。

```
Agent Teams:
  ┌─ learnings-researcher ─┐
  │       (並列 spawn)      │──▶ scout ──▶ designer ──▶ planner
  └─ analyst ──────────────┘
```

```markdown
## Step 1: チーム作成
spawnTeam: "planning"

## Step 2: 並列 Phase (0.5 + 1)
# 同時に spawn — 互いに独立しているので並列実行可能
spawnTeammate: learnings-researcher  # 過去の学び検索
spawnTeammate: analyst               # 要件定義

## Step 3: 順次 Phase (1.5 → 2 → 3)
# Phase 0.5 + 1 完了後に spawn
spawnTeammate: scout     # requirements.md を読んでギャップ分析
# scout 完了後
spawnTeammate: designer  # アーキテクチャ設計
# design.md 完了後
spawnTeammate: planner   # タスク分解
```

**ポイント**: 並列 spawn できるフェーズを見極める。依存関係がないなら同時に spawn して時間を短縮。

### パターン3: 大規模並列 + 自律調整（/ultrawork）

5+ teammates が共有タスクリストで自律的に作業する。オーケストレーターの本領。

```markdown
## Step 1: チーム作成
spawnTeam: "ultrawork"

## Step 2: タスク登録
TaskCreate で全タスクを登録、TaskUpdate で依存関係を設定

## Step 3: Teammates Spawn
orchestration.yml の各タスクグループに対して teammate を spawn
各 teammate の spawn prompt に:
- agents/{agent-name}.md の指示
- 担当タスクの詳細
- 参照すべき standards
- 「完了したら Lead にメッセージ、問題は他の teammate にも共有」

## Step 4: 監視・調整
Lead は定期的に:
- teammates の進捗をメッセージで確認
- 問題があればリダイレクト
- 完了した teammate に追加タスクを割り当て

## Step 5: 検証・完了
code-reviewer teammate で最終レビュー → DONE
```

**ポイント**: TaskCreate/TaskUpdate と TeammateTool の組み合わせが核心。ここで重要な注意点がある。

### 混同注意: Task tool と TaskCreate は別物

名前が似ていて混乱する。

| ツール | 用途 | Agent Teams での扱い |
|--------|------|---------------------|
| **Task tool** | subagent を spawn する | TeammateTool に**置換**された |
| **TaskCreate/TaskUpdate/TaskList** | 共有タスクリストの管理 | Agent Teams の**中核** |

Task tool は廃止対象。TaskCreate 系は Agent Teams でむしろ重要度が上がった。

## ディスパッチ戦略: 全部 Agent Teams でいいのか？

全てを Agent Teams にするとトークンコストが上がる。しかし、質を重視するなら Agent Teams を積極的に使う方がいい。

o-m-cc ではタスク規模で切り替えるルールを設けた。

### 判断フロー

```
タスクを受け取る
  │
  ├─ Glob/Grep 1回で答えが出る？ → 【S】直接実行
  │
  ├─ 判断・分析が必要？ ────────→ 【M】Agent Teams (2-3 teammates)
  │
  └─ 複数工程・並列作業？ ──────→ 【L】Agent Teams (5+ teammates)
```

**迷ったら Agent Teams を選ぶ。** トークンに余裕があるなら質を取る。

| 規模 | 例 | 方式 |
|------|-----|------|
| **S** | 「このクラスどこ？」 | Lead が直接 Grep |
| **M** | 「このバグの原因調べて」 | debugger × 2 で仮説競合 |
| **M** | 「レビューして」 | code-reviewer + security-reviewer |
| **L** | 「認証機能を実装して」 | 5+ teammates + TaskCreate |

### 相乗効果パターン

Agent Teams の真価は「組み合わせ」。1+1 が 3 になる。

| パターン | 組み合わせ | 効果 |
|---------|-----------|------|
| **相互レビュー** | code-reviewer + security-reviewer | 互いの発見を共有・補完 |
| **仮説競合** | debugger × 2-3 | 異なる仮説を並列検証、偏りを排除 |
| **多角調査** | explore + researcher + analyst | コード・外部情報・要件を同時調査 |
| **設計批評** | designer + critic | 設計しながらリアルタイムレビュー |
| **実装+品質** | frontend + code-reviewer | 実装しながら逐次レビュー |

**仮説競合パターン**が特に強力:

```
debugger-1: 「認証トークンの期限切れが原因」
debugger-2: 「いや、CORS 設定の問題。トークンは正常だった」
→ 議論して真因を特定
```

1人だと最初に見つけた仮説に引っ張られる（アンカリング）。複数で競合させると、生き残った仮説が正解である確率が上がる。これは[公式ドキュメント](https://code.claude.com/docs/en/agent-teams)でも推奨されている。

## hooks で Agent Teams を自動化する

Agent Teams 単体でも強力だが、Claude Code の hooks と組み合わせるとワークフローが自動化できる。

### 2.1.33 の新 hook イベント

リリース翌日の 2.1.33 で Agent Teams 専用の hook イベントが追加された。

| イベント | 発火タイミング | 用途 |
|---------|-------------|------|
| **TeammateIdle** | teammate が作業完了して idle になった | 残タスクの再割り当て |
| **TaskCompleted** | タスクが completed になった | 進捗表示、依存タスクのアンブロック |

### o-m-cc での実装

```json
// hooks.json
{
  "TeammateIdle": [{
    "matcher": "",
    "hooks": [{
      "type": "command",
      "command": "bash ${CLAUDE_PLUGIN_ROOT}/hooks/teammate-idle.sh",
      "timeout": 5000
    }]
  }],
  "TaskCompleted": [{
    "matcher": "",
    "hooks": [{
      "type": "command",
      "command": "bash ${CLAUDE_PLUGIN_ROOT}/hooks/task-completed.sh",
      "timeout": 5000
    }]
  }]
}
```

`teammate-idle.sh` の動作:

```bash
# 残タスクをカウント
REMAINING=$((TOTAL_TASKS - COMPLETED_TASKS))

if [[ $REMAINING -gt 0 ]]; then
  # 残タスクあり → systemMessage で再割り当てを示唆
  jq -n '{
    "systemMessage": "💤 teammate が idle。残タスク X件 — 割り当てを検討してください。"
  }'
else
  # 全タスク完了 → 最終レビューを促す
  jq -n '{
    "systemMessage": "✅ 全タスク完了。code-reviewer で最終レビューを。"
  }'
fi
```

`task-completed.sh` は進捗（X/X, XX%）を表示し、依存タスクのアンブロックを通知する。

### hooks 全体像

o-m-cc の hooks 構成（Agent Teams 対応含む）:

| イベント | Hook | 説明 |
|---------|------|------|
| SessionStart | `resume-session.sh` | 前回セッション復元 |
| Stop | `stop-guard.sh` | Sisyphus ガード（レビュー確認） |
| UserPromptSubmit | `focus-guard.sh` | タスク進行中の脱線防止 |
| PreToolUse | `security_reminder_hook.py` | セキュリティパターン検出 |
| PostToolUse | `auto-verify.sh` | フェーズ完了時の自動検証 |
| **TeammateIdle** | `teammate-idle.sh` | idle teammate への再割り当て |
| **TaskCompleted** | `task-completed.sh` | 進捗・アンブロック通知 |

hooks と Agent Teams を組み合わせることで「teammate が idle → 残タスク割り当て → 完了 → 進捗表示 → 全完了 → 最終レビュー」が自動で回る。

## agent memory: エージェントに記憶を持たせる

2.1.33 で追加された `memory` frontmatter。エージェントに永続メモリを持たせられる。

```yaml
# agents/code-reviewer.md
---
name: code-reviewer
tools: Read, Glob, Grep, Bash, Write
model: sonnet
memory: project
---
```

`memory: project` を設定すると、そのエージェントはプロジェクト固有の知見を永続的に記憶する。

### どのエージェントに記憶を持たせるか

**記憶あり（`memory: project`）** — 判断・学習が蓄積するエージェント:

| エージェント | 記憶する内容 |
|-------------|------------|
| code-reviewer | プロジェクト固有のコードパターン・規約 |
| security-reviewer | セキュリティ判断の履歴 |
| designer | 過去の設計判断 |
| advisor | 過去の相談・戦略判断 |
| planner | タスク分解パターン |
| analyst | 要件分析の知見 |
| frontend | UI規約・コンポーネントパターン |
| debugger | 過去のバグパターン |

**記憶なし** — 読み取り専用・短命のエージェント:

explore, scout, critic, researcher, learnings-researcher, vision, document-writer, code-simplifier

基準は「前回のプロジェクト作業の知見が、次回の作業品質を向上させるか」。code-reviewer がプロジェクト固有の規約を覚えていれば、毎回同じ指摘をしなくて済む。explore は毎回新鮮に検索する方が正確。

## 内部構造

オーケストレーターを自作する人向けに、ファイル構造も共有しておく。

### Agent Teams のファイル配置

```
~/.claude/
├── teams/{team-name}/
│   └── config.json          # チーム設定（メンバー情報）
└── tasks/{session-id}/
    ├── .lock                # ファイルロック（競合防止）
    ├── .highwatermark       # タスクID採番
    ├── 1.json               # タスク #1
    ├── 2.json               # タスク #2
    └── ...
```

### タスク JSON

```json
{
  "id": "1",
  "subject": "認証モジュールのリファクタリング",
  "description": "JWT → セッションベースに変更",
  "activeForm": "認証モジュールをリファクタリング中",
  "status": "in_progress",
  "blocks": ["3"],
  "blockedBy": []
}
```

`blocks` / `blockedBy` で依存関係を表現。`.lock` で teammate 間のクレーム競合を防止。

### o-m-cc のプラグイン構造

```
o-m-cc/
├── agents/                    # エージェント定義（teammate spawn 時に参照）
│   ├── capabilities.md        # ディスパッチ戦略 + 能力サマリー
│   ├── code-reviewer.md       # memory: project
│   ├── security-reviewer.md   # memory: project
│   └── ...（16エージェント）
├── commands/                  # スラッシュコマンド
│   ├── plan.md                # Agent Teams オーケストレーター
│   ├── review.md              # Agent Teams 並列レビュー
│   └── ultrawork.md           # Agent Teams 大規模並列
└── hooks/                     # 自動化
    ├── hooks.json
    ├── teammate-idle.sh       # TeammateIdle
    ├── task-completed.sh      # TaskCompleted
    └── ...（10 hooks）
```

コマンドの frontmatter で `allowed-tools: TeammateTool` を指定し、コマンド本文でオーケストレーションの手順を自然言語で記述する。エージェント定義は `agents/{name}.md` に分離し、spawn prompt から参照する構造。

## 制限事項

Agent Teams はまだ experimental。オーケストレーターを作る際に知っておくべき制限:

- **セッション復元不可**: `/resume` で in-process teammates は復元されない
- **1チーム/セッション**: cleanup してから新チームを作る必要あり
- **ネスト不可**: teammate が自分の teammate を spawn できない
- **Lead 固定**: チーム作成者が永久に Lead
- **権限は spawn 時に固定**: 全 teammate が Lead の権限を引き継ぐ
- **split-pane は tmux/iTerm2 必須**: VS Code ターミナルでは使えない

## まとめ

既存のマルチエージェントワークフローを Agent Teams で強化するポイント:

1. **spawn prompt に「他 teammate への共有」を明記** — これがないと subagent と変わらない
2. **並列 spawn できるフェーズを見極める** — 依存関係がないものは同時 spawn
3. **TaskCreate で共有タスクリスト** — teammates の自律調整の基盤
4. **hooks で自動化** — TeammateIdle / TaskCompleted でワークフローが回る
5. **memory: project で知見を蓄積** — 使うほど賢くなるエージェント
6. **ディスパッチ戦略で使い分け** — S は直接、M/L は Agent Teams

o-m-cc はまだ主に自分用だが、マルチエージェントオーケストレーターを組んでいる人の参考になればと思い公開している。

GitHub: [kok1eee/o-m-cc](https://github.com/kok1eee/o-m-cc)

## 参考

- [Orchestrate teams of Claude Code sessions](https://code.claude.com/docs/en/agent-teams) - 公式ドキュメント
- [Introducing Claude Opus 4.6](https://www.anthropic.com/news/claude-opus-4-6) - Anthropic 公式アナウンス
- [Claude Code's Hidden Multi-Agent System](https://paddo.dev/blog/claude-code-hidden-swarm/) - TeammateTool の内部構造分析
- [Anthropic releases Opus 4.6 with new 'agent teams'](https://techcrunch.com/2026/02/05/anthropic-releases-opus-4-6-with-new-agent-teams/) - TechCrunch
