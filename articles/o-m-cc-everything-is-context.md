---
title: "Claude Codeと並走して3ヶ月。手探りで作ったものに全部名前がついた"
emoji: "🧠"
type: "tech"
topics: ["claudecode", "ai", "plugin", "multiagent"]
published: false
---

:::message
**TL;DR** — Claude Code のアップデートに追従し続けて o-m-cc を v0.19.1 まで育てた。`/simplify` にハマって `/quality-gate` を作り、論文 "Everything is Context" で手探りの実装に全部名前がついた。
:::

## はじめに

> 業務自動化Pythonエンジニア。バイブコーディング歴1年 ≒ エンジニア歴。

Claude Code を3ヶ月使い続けて、プラグイン [o-m-cc](https://github.com/kok1eee/o-m-cc) を作った。12のエージェントと hooks による自動化を持つマルチエージェントシステム。v0.1 から v0.19.1 まで、Claude Code のアップデートに追従し続けている。

この記事は3つの話をする：

1. Claude Code のアップデートが面白い話
2. `/simplify` にハマって `/quality-gate` を作った話
3. 手探りで作ったものに、論文で名前がついた話

前回の記事:

https://zenn.dev/kok1eee/articles/o-m-cc-claude-code-2149

## Claude Code のアップデートが面白い

Claude Code は進化が速い。数日おきにリリースがあり、そのたびにエージェントシステムの可能性が広がる。

o-m-cc はその都度対応してきた。Changelog が物語っている：

| バージョン | 対応した Claude Code 機能 |
|-----------|-------------------------|
| v0.15 | Faceted Prompting、集約ロジック（TAKT から着想） |
| v0.16 | `settings.json` 同梱、`background: true`、`isolation: worktree` |
| v0.17 | Progressive Disclosure（段階的開示） |
| v0.18 | CONTEXT.md 3層アーキテクチャ、`/simplify` 統合 |
| v0.19 | `/quality-gate`、`agent_type` ガード、`continue:false` |

いくつか印象的だったものを挙げる。

### Agent Teams（TeammateTool）

最大の転換点。エージェント同士が peer-to-peer で協調できるようになった。「中央オーケストレーターなし」の設計が可能になり、o-m-cc の根幹になった。

### `isolation: worktree`

エージェントがファイル変更する際に worktree で分離される。並列実行時のファイル競合がなくなった。

### `/simplify`（2.1.63）

組み込みのコード品質チェック。再利用性・品質・効率を自動でレビューして修正してくれる。これが後の話につながる。

### `agent_type` / `continue:false`（2.1.69）

サブエージェント（teammate）を識別できるようになった。stop-guard で「メインエージェントの完了だけチェックして、teammate の完了は無視する」が実装できた。`continue:false` で idle な teammate を明示的に停止できるようになった。

アップデートのたびに「あ、これで前から困ってたことが解決できる」となる。追従は大変だが楽しい。

## /simplify にハマった

`/simplify` が 2.1.63 で追加されてから、毎回使うようになった。

コードを書いたあとに `/simplify` を走らせると、冗長な箇所や改善点を見つけて直してくれる。地味だが確実に品質が上がる。

問題は、手動で毎回 `/simplify` → `/review` を実行するのが面倒だったこと。

### /quality-gate を作った

そこで `/quality-gate` スキルを作った。

```
/quality-gate
  Step 1: /simplify（再利用・品質・効率の自動レビュー+修正）
  Step 2: diff 確認
  Step 3-5: Review Council（code-reviewer + security-reviewer + critic 並列）
  Step 6: Critical あれば自動修正 → Step 3 に戻る
  Step 7: 静的解析（ruff/ty/shellcheck）
```

`/simplify` → Review Council → 静的解析を一発で実行するパイプライン。

### proof マーカーで強制する

さらに **proof マーカー** という仕組みを入れた：

```
/quality-gate 完了時 → <proof>QUALITY_GATE_PASSED</proof> を出力
stop-guard（Stop hook） → proof マーカーがなければ DONE を許可しない
```

「品質ゲートを通さないと完了できない」を仕組みで強制する。Sisyphus（止まらない）哲学の延長。

### 静的解析の順序問題

最初は静的解析を Step 1 に置いていた。でも考えてみれば、`/simplify` や Review Council で修正が入る**前**のコードを解析しても意味がない。最終成果物に対して走らせないと。

修正して Step 7（最後）に移動した。こういう地味な調整の積み重ね。

## 手探りで作ったものに名前がついた

ここまでアップデート追従と `/quality-gate` の話をしてきたが、もう一つ大きな出来事があった。

["Everything is Context"（arXiv:2512.05470）](https://arxiv.org/abs/2512.05470) という論文を読んだら、自分のフォルダ構造が学術用語に翻訳された。

### 対応表

| 論文の概念 | 説明 | o-m-cc での実装 |
|-----------|------|----------------|
| **Scratchpad** | 一時作業区。現在のタスクの中間状態 | `.claude/context.md`, `plan/` |
| **Fact Memory** | プロジェクト固有の事実記憶 | `MEMORY.md`, `.claude/agent-memory/` |
| **Experiential Memory** | 複数プロジェクトを横断する経験知 | `facets/references/`, auto-memory |
| **Context Constructor** | トークン予算内でコンテキストを選択的にロード | Progressive Disclosure（3層構造） |
| **Bounded Reasoning** | 有限なコンテキスト窓での戦略的な取捨選択 | `rules/` 自動ロード vs `docs/` 按需ロードの分離 |
| **Context Evaluator** | ロードしたコンテキストの妥当性を検証 | hooks（stop-guard, auto-verify, memory-digest） |

全部、先に実装があって、後から名前を知った。

### 一番痛かった坑：トークン爆発

最初は `rules/` に全情報を詰め込んでいた。エージェント定義も、レビュー基準も、セキュリティチェックリストも。

結果、コンテキストが爆発した。エージェントは肝心の指示を読み落とし、的外れな回答を返す。

対策として**3層に分離**した：

```
Layer 1: frontmatter（常時ロード）    → ~60 tokens/agent
Layer 2: 本文（エージェント起動時）     → ~100-200 tokens/agent
Layer 3: facets/references/（必要時に Read） → ~200-500 tokens/file
```

全情報を常時ロードする場合と比べて、通常時のトークン消費が約 1/3 になった。

論文はこれを **Context Constructor** と呼んでいる。「トークン予算という有限リソースの中で、タスクの優先度に基づいてコンテキストを動的に配分する仕組み」。

### 唯一の隙間：Context Evaluator

論文との対応表を作ったとき、一つだけ「隙間」が見えた。

o-m-cc は**出力の検証**（stop-guard がタスク完了を検証、auto-verify がテスト通過を確認）はしていたが、**入力コンテキストの鮮度チェック**はしていなかった。

例えば `MEMORY.md` に「`hooks/old-script.sh` のパターンは〜」と書いてあっても、そのファイルがもう削除されていたら？古い前提で作業を始めてしまう。

対応として、既存の `memory-digest.sh`（SessionStart hook）に20行追加した：

```bash
# MEMORY.md 内のファイルパス参照の実在チェック
while IFS= read -r ref_path; do
  if [[ ! -e "${CWD}/${ref_path}" ]]; then
    STALE_REFS+=("$ref_path")
  fi
done < <(grep -oE '`[a-zA-Z0-9_./-]+\.(md|sh|py|json|yaml|yml|toml|ts|js|txt)`' \
  "$PROJECT_MEMORY_FILE" 2>/dev/null | tr -d '`' | sort -u)
```

初回実行で即座に検知した：

```
🔍 MEMORY.md 古い参照（ファイルが存在しません）
   - plugin.json
   → 該当する記述を確認・更新してください
```

`plugin.json` の正確なパスは `.claude-plugin/plugin.json` だが、MEMORY.md では短縮形で書かれていた。論文の隙間を埋めたら、実問題が見つかった。

## 実践在前、命名在後

「Context Constructor を実装しよう」と思って作ったのではなく、「トークンが爆発して困ったから分離した」結果が Context Constructor だった。

「Bounded Reasoning を考慮しよう」と思ったのではなく、「エージェントが指示を読み落とすから自動ロードを減らした」結果が Bounded Reasoning への対処だった。

痛みから始めて、実験で形にして、論文で名前を知る。この順序だからこそ、概念が腹落ちする。

## まとめ

Claude Code と並走してプラグインを作り続けていると、3つのことが同時に起きる：

1. **アップデートで新しい可能性が生まれる** — `agent_type` で teammate を識別、`/simplify` で品質チェック自動化、worktree で並列安全
2. **実践から仕組みが生まれる** — 毎回 `/simplify` を手動で走らせるのが面倒 → `/quality-gate` として自動化、proof マーカーで強制
3. **手探りの実装に名前がつく** — トークン爆発の対策が Context Constructor、古い参照の検知が Context Evaluator

これは LLM の token window という物理的制約から必然的に導かれる問題で、誰がやっても似た解に収束する。論文 ["Everything is Context"](https://arxiv.org/abs/2512.05470) はその収束先に名前をつけてくれた。

先に踏んだ坑に名前がつく快感は格別だ。

GitHub: [kok1eee/o-m-cc](https://github.com/kok1eee/o-m-cc)
