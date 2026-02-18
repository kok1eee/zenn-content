---
title: "o-m-cc v0.11-v0.14 — 引き算で強くなるAIエージェントプラグイン"
emoji: "🗑️"
type: "tech"
topics: ["claudecode", "ai", "plugin", "automation", "refactoring"]
published: true
---

## はじめに

> 業務自動化Pythonエンジニア。バイブコーディング歴1年 ≒ エンジニア歴。

自作の Claude Code プラグイン [o-m-cc](https://github.com/kok1eee/o-m-cc) を v0.10 から v0.14 まで一気に進めた。4バージョン分のアップデートだが、やったことの大半は**削除**。

前回の記事で Council パターンの導入（v0.10）を書いた。その後書きで v0.12 の大掃除にも触れたが、v0.11〜v0.14 の全体を通して見ると「引き算」という一貫したストーリーがある。

https://zenn.dev/kok1eee/articles/o-m-cc-council-pipeline-hybrid

https://zenn.dev/kok1eee/articles/o-m-cc-claude-code-plugin

## 4バージョンの要約

| ver | 何を捨てたか | 何が残ったか |
|-----|-------------|-------------|
| 0.11 | `learned/` ディレクトリ | HANDOVER.md の VCS 履歴 |
| 0.12 | `spec/` ディレクトリ | CLAUDE.md 一本 |
| 0.13 | 手動のスキル昇格提案 | 自動スキル昇格 |
| 0.14 | `commands/*.md` フラット構造 | `skills/*/SKILL.md` ディレクトリ構造 |

足したものより捨てたものの方が多い。結果として -1,872 行。

## v0.11: learned/ を捨てて VCS に任せる

### Before

学びを記録する専用ディレクトリがあった。

```
spec/standards/learned/
├── 2026-01-15-session-based-auth.md
├── 2026-01-18-flaky-test-fix.md
└── ...
```

`/learn` コマンドで手動記録し、`pattern-detector.sh` hook で自動検出も試みた。

### 問題

- 手動で `/learn` を呼ぶ習慣がつかない
- 自動検出は誤検出が多い
- 結局読み返す機会がない
- HANDOVER.md にも同じことが書いてある

### After

`learned/` を丸ごと削除。代わりに **HANDOVER.md を VCS 管理対象にした**。

```
セッション終了
  → generate-handover.sh が HANDOVER.md を自動生成
  → VCS にコミット
  → 次のセッションで diff 履歴として蓄積
```

HANDOVER.md はセッションごとに上書きされるが、VCS の diff 履歴には過去の意思決定・教訓が全て残る。専用ディレクトリを管理するより、**既にある仕組み（VCS）に乗せる方がうまくいく**。

`learnings-researcher` エージェントの検索対象も `learned/` → HANDOVER.md VCS 履歴に変更。

## v0.12: spec/ を捨てて CLAUDE.md 一本に

前回の記事の後書きで詳しく書いたので要点だけ。

### 何が起きたか

`spec/` に技術規約（standards）やプロジェクト文脈（steering）を置いていたが、中身は CLAUDE.md と重複していた。Claude Code の設計思想的にも **CLAUDE.md がプロジェクトの唯一の知識ソース**であるべき。

```
# Before
spec/
├── standards/   # 技術規約 → CLAUDE.md と重複
├── steering/    # プロジェクト文脈 → CLAUDE.md と重複
├── rules/       # Sisyphus ルール → Default Agent に移動済み
└── plan/        # 計画ファイル → これだけ残す

# After
plan/            # ルートに昇格
CLAUDE.md        # 唯一のソース
```

エージェント 16 → 14、コマンド 10 → 7。`spec/` ディレクトリ自体が消えた。

## v0.13: 手動提案を捨てて自動昇格に

### Before

`promote-checker.sh`（Stop hook）が HANDOVER.md の VCS 履歴を検索して、繰り返しパターンを見つけたら「`/promote` でスキル昇格を検討してください」と**提案**していた。

しかし提案されても実行しない。セッション終了時に「こんなパターンがありますよ」と言われても、次のセッションでは忘れている。

### After

提案ではなく、**自動で昇格を実行する**ようにした。

```
promote-checker.sh（Stop hook）
  → VCS 履歴を横断検索
  → 繰り返しパターンを検出
  → 自動でスキル昇格を実行
      ├─ 複数プロジェクトで出現 → ~/.claude/CLAUDE.md（グローバルルール）
      └─ 単一プロジェクトで頻出 → CLAUDE.md（プロジェクトルール）
```

さらに `~/.claude/skill-candidates.md` にクロスプロジェクトの候補を蓄積する仕組みも追加。プロジェクト A で学んだことがプロジェクト B でも再現したら、グローバルルールに昇格する。

### /promote もクロスプロジェクト対応

手動実行の `/promote` も刷新:

- `~/.claude/skill-candidates.md`（クロスプロジェクト候補）
- HANDOVER.md VCS 履歴（ローカル候補）

両方を統合して候補を提示。昇格先も4択に拡張:

1. エージェント定義に追加
2. スキル（スラッシュコマンド）として作成
3. プロジェクトルール（CLAUDE.md）
4. グローバルルール（`~/.claude/CLAUDE.md`）

## v0.14: フラット構造を捨ててディレクトリに

### Before

スラッシュコマンドは `commands/` にフラットに置いていた。

```
commands/
├── plan.md
├── review.md
├── handover.md
├── init.md
├── install.md
├── audit.md
└── promote.md
```

### 問題

Claude Code のスキルシステムは `skills/*/SKILL.md` というディレクトリ構造を前提にしている。フラットファイルだと将来スキルごとにリソースファイルを追加したくなったとき困る。

### After

```
skills/
├── plan/SKILL.md
├── review/SKILL.md
├── handover/SKILL.md
├── init/SKILL.md
├── install/SKILL.md
├── audit/SKILL.md
└── promote/SKILL.md
```

合わせて `disable-model-invocation: true` を install/init/audit/promote の4スキルに追加。これらは明示的に呼ぶべきコマンドなので、AI が勝手に発動しないようにした。

自動発動するのは plan（「計画して」）、review（「レビューして」）、handover（「引き継ぎ」）の3つだけ。

## 引き算の判断基準

4バージョンの整理を通して見えてきた基準:

### 1. 二重管理になっていないか

```
learned/ と HANDOVER.md → learned/ を削除
spec/standards/ と CLAUDE.md → spec/ を削除
promote の提案と手動実行 → 提案を削除、自動実行に
```

同じ情報が2箇所にあるなら、片方を消す。残す方は「既に動いている仕組み」を選ぶ。

### 2. 手動の習慣がつくか

```
/learn で手動記録 → 習慣がつかない → 自動化（HANDOVER.md VCS）
/promote を手動実行 → 忘れる → 自動化（promote-checker）
```

「手動で呼んでください」は機能しない。人間は忘れる。hooks に乗せて自動化するか、消す。

### 3. 名前が実態を反映しているか

```
spec/ → 仕様を管理していない → plan/ に改名
commands/ → Claude Code のスキルシステムに合わない → skills/ に移行
```

名前と中身がズレたら、コードの可読性と同じで、認知負荷になる。

## まとめ

v0.11-v0.14 の変更量:

| 指標 | Before (v0.10) | After (v0.14) |
|------|----------------|---------------|
| ディレクトリ | spec/, learned/, commands/ | plan/, skills/ |
| プロジェクト知識のソース | 3箇所 | CLAUDE.md 一本 |
| スキル昇格 | 手動提案 | 自動実行 |
| 行数 | 多い | **-1,872行** |

AI エージェントプラグインに限らず、ソフトウェアは足すより引く方が難しいし、引いた後の方が強い。不要なものを消すと、残ったものの意図が明確になる。

次の v0.15.0 では逆に「足す」方向のアップデートをした。外部プロジェクトの設計思想に触発されて、新しい概念を3つ導入した。それは次の記事で。

GitHub: [kok1eee/o-m-cc](https://github.com/kok1eee/o-m-cc)
