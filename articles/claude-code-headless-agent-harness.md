---
title: "claude -p が大袈裟すぎるので、非対話エージェント専用のハーネスを探した話"
emoji: "🔧"
type: "tech"
topics: ["claudecode", "ai", "agent", "slackbot"]
published: false
---

## はじめに

> 業務自動化Pythonエンジニア。バイブコーディング歴1年 ≒ エンジニア歴。

前回・前々回の記事で、`claude -p` を使ったナレッジ Bot とアンビエントエージェントの話を書いた。

https://zenn.dev/tazawa_masayoshi/articles/knowledge-chatbot-agentic-search

`claude -p` は便利だ。`--allowedTools` でツール制限、`--max-turns` で暴走防止、`--append-system-prompt` でコンテキスト注入。これだけで非対話エージェントが動く。

でも使い倒していくうちに、ひとつ気になり始めた。**`claude -p` はコーディングエージェントのハーネスであって、非対話エージェント専用のハーネスではない**。

## `claude -p` の何が大袈裟か

`claude -p` は Claude Code のヘッドレスモード。つまり Claude Code の全機能がそのまま動く。

ナレッジ Bot に必要なのは `Read`、`Grep`、`Glob` の3つだけ。でも `claude -p` を起動すると、裏では CLAUDE.md の読み込み、プラグイン・スキルの展開、hooks の実行が走る。コーディングエージェントとして必要な初期化が全部動く。

ナレッジの検索・回答だけさせたいのに、毎回コーディングエージェントとしてフル装備で起動している。

## o-m-cc の HEADLESS フラグで対処していた

自作プラグイン [o-m-cc](https://github.com/kok1eee/o-m-cc) では、`CLAUDE_HEADLESS=1` という環境変数を用意している。

```bash
# CLAUDE_HEADLESS=1 が設定されている場合、hook をスキップ
# 用途: claude -p で ops 実行時にコンテキスト注入やブロックを防止
is_headless() {
  [[ "${CLAUDE_HEADLESS:-}" = "1" ]]
}
```

`claude -p` を subprocess から呼ぶとき、`CLAUDE_HEADLESS=1` を環境変数に渡す。するとプラグインの hooks がスキップされて、余計な処理が走らない。

対処療法としては動く。でも根本的に、`claude -p` 自体がコーディングエージェント用のハーネスであることは変わらない。**不要な機能を削ぎ落とした、非対話エージェント専用のハーネスが欲しい。**

## pi-mono を試した → コーディングエージェントだった

[pi-mono](https://github.com/badlogic/pi-mono) を知った。bash、read、edit、write の4ツールだけのミニマリスト設計。システムプロンプトは1000トークン以下。Claude Code から不要なものを削ぎ落としたやつだ。

期待して触ってみたが、これは **opencode と同じカテゴリ**だった。Claude Code の軽量代替であって、非対話エージェント用ではない。対話的にコードを書くためのツール。自分が欲しかったのとは違う。

## claude-agent-sdk-pi → 結局 `claude -p` 経由

次に [claude-agent-sdk-pi](https://github.com/prateekmedia/claude-agent-sdk-pi) を見つけた。Claude Agent SDK を pi に統合するアプローチ。SDK 経由で Claude のモデルを呼べるなら、`claude -p` を経由せずに済むのでは？

でも実際には、Claude Code のログインか API キーで認証して、ツール呼び出しは `claude -p` が処理する構造だった。**`claude -p` の上にもう一層被せただけ**で、根本的な問題は何も変わらない。本末転倒。

## Bedrock API で直接叩いてみた

方向を変えた。`claude -p` を使わず、Bedrock API で Anthropic のモデルを直接呼ぶ。

Sonnet 4.6 を使ったら、**1ラリーで $0.7**。ナレッジ検索で複数ファイルを読むとトークン消費がリッチすぎる。1日20件の問い合わせで月額 $400 超。Max プランの方が安い。

じゃあ Haiku 4.5 なら？コストは下がる。でも **エージェンティックに動いてくれない**。

```
Haiku: Grep しますか？ → 承認待ち
自分: やれ
Haiku: 3ファイル見つかりました。読みますか？ → 承認待ち
自分: 読め
Haiku: 回答を生成しますか？ → 承認待ち
```

聞く、止まる、聞く、止まる。ナレッジ Bot に必要なのは「質問を受けたら自分で Grep して Read して回答を返す」こと。Haiku はツールを使う判断を自力でできず、毎回確認を求めてくる。

やっぱり **Sonnet 4.6 のパワーが必要**。Sonnet は自分で検索戦略を立てて、複数ファイルを読んで、統合して回答を作る。このエージェンティックな動きができるのは Sonnet のモデル性能あってこそ。

でも Bedrock で Sonnet は高すぎる。`claude -p` は大袈裟すぎる。詰んだ。

## free-code / not-claude-code-emulator

詰んだと思ったところで、[not-claude-code-emulator](https://github.com/code-yeongyu/not-claude-code-emulator) の存在を知った。Claude Code の API リクエストをエミュレートするツール。[free-code](https://github.com/paoloanzn/free-code) も同じ文脈で、3,500+ star で完全に公知のプロジェクト。

Claude Code のハーネス（ツール制限、セッション管理、暴走防止）を、Max プランの `claude -p` に依存せず再現できる。バックエンドに Anthropic API を直接指定できる。

そのままでは動かなかった。が、調整して**実際に動かすことができた**。

Sonnet 4.6 のエージェンティックな動きを、`claude -p` の大袈裟なハーネスなしで使える。コストは API 課金だが、不要な初期化やプラグイン展開が走らない分、トークン消費が抑えられる。

## これで何ができるか

`claude -p` から解放されると、ハーネスを自分で設計できるようになる。

### MCP サーバーを自由に接続できる

`claude -p` では Claude Code が読み込む MCP 設定に縛られる。普段のコーディング作業では不要な MCP サーバーを、エージェント専用に設定できる。

例えば [Serena](https://github.com/oraios/serena)。セマンティック検索とシンボルレベルのコード理解を提供する MCP サーバー。ナレッジ Bot にこれを接続したら、`Grep` のテキスト検索だけでなく、意味ベースの検索ができるようになる。検索品質が上がる可能性がある。

### 専用スキルを盛り盛りにできる

`claude -p` + `--append-system-prompt` ではシステムプロンプトに全部詰め込む必要があった。専用ハーネスなら、用途ごとにスキルファイルを分離して、必要なものだけ読み込める。

ナレッジ Bot 用、定型作業用、コードレビュー用。それぞれに最適化したスキルセットを用意できる。`claude -p` の「何でも入り」ではなく、**用途に特化したエージェント**を作れる。

## あとがき

`claude -p` から始まって、一周回った。

最初は `claude -p` の便利さに乗って、subprocess 一行でナレッジ Bot もアンビエントエージェントも動かしていた。でも使い倒すほどに「これはコーディングエージェントのハーネスであって、自分が作りたいものとは微妙にズレている」と感じるようになった。

pi-mono は非対話用じゃなかった。SDK 経由は本末転倒だった。Bedrock API は高かった。Haiku は賢さが足りなかった。

回り道をしたが、最終的に**Sonnet 4.6 のエージェンティック性能を、軽量なハーネスで使える環境**にたどり着いた。

前回の記事で「ハーネスに乗れ」と書いた。今回の学びはこうだ。

**乗るハーネスは選べ。合わなければ乗り換えろ。**
