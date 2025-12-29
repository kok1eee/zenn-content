---
title: "Claude Code × jujutsu - AIがバージョン管理を実行する時代"
emoji: "🤖"
type: "tech"
topics: ["jujutsu", "claudecode", "ai", "versioncontrol", "git"]
published: false
---

## はじめに

> 業務自動化Pythonエンジニア。バイブコーディング歴 = エンジニア歴。
> 過去に`git stash`でデータ消失して泣いた経験あり。

「`git add`して`git commit`して...」

この呪文、何回唱えた？俺は数えきれない。

そして`git stash`したまま忘れて、気づいたら消えてた。あの絶望感、わかる人にはわかるはず。

今、俺は**84プロジェクトのモノレポ**をClaude Code（AIアシスタント）と一緒に開発してる。で、コミットやプッシュするのは**俺じゃなくてClaude**なんだよね。

「え、AIにバージョン管理させて大丈夫？」って思うでしょ。

**大丈夫。jujutsuならね。**

## なぜjujutsuなのか

### Gitの問題、正直キツくない？

AIにGit使わせると、まあ事故る。

1. **`git add`忘れる** → 「あれ、変更入ってない...」
2. **`git reset --hard`の破壊力** → 取り返しつかない
3. **コンフリクト地獄** → 手動で直すしかない

人間でもミスるのに、AIならなおさら。

### jujutsuが全部解決してくれた

```bash
# Gitの場合（2ステップ、ミスりやすい）
git add .
git commit -m "feat: 新機能"

# jujutsuの場合（1ステップ、変更は勝手に追跡される）
jj describe -m "feat: 新機能"
```

これ、マジで革命。

- **`git add`いらない** → ファイル変更したら勝手に追跡される
- **`jj undo`で何でも戻せる** → ミスっても平気。何度でもやり直せる
- **Git完全互換** → 既存のGitリポジトリでそのまま使える

**ミスっても`jj undo`で戻せる。この安心感、半端ない。**

## Claude Codeにjjを教える

### ルールファイルに書くだけ

Claude Codeは`~/.claude/rules/`のマークダウンを読んで、その通りに動く。

**~/.claude/rules/jj.md**
```markdown
# Jujutsu (jj) Rules

## 基本
- **gitコマンドは使わない** → `jj`を使用
- コミットメッセージ: Conventional Commits形式

## 主要コマンド
jj status              # 状態確認
jj diff                # 差分表示
jj describe -m "msg"   # コミットメッセージ設定
jj new                 # 新しい変更セット作成
jj bookmark set main -r @  # ブックマーク設定
jj git push            # リモートにプッシュ
```

これだけ。たったこれだけで、Claudeは「gitじゃなくてjj使うのね」って理解してくれる。

### CLAUDE.mdにも念押し

プロジェクトルートの`CLAUDE.md`にも書いとく：

```markdown
## 基本ルール
- **バージョン管理はJujutsu (jj)** を使用（gitコマンドは使わない）
- **コミットはConventional Commits**形式（`feat:`, `fix:`, `chore:`等）
```

念には念を。

## 俺のエイリアス設定、公開する

### ~/.jjconfig.toml

```toml
[aliases]
# fetch + rebase を一発で。これめっちゃ使う
pull = ["util", "exec", "--", "sh", "-c", "jj git fetch && jj rebase -d main"]

# 状態確認（stは打ちやすい）
st = ["status"]
l = ["log", "-n", "10"]

# mainから新しい作業開始
fresh = ["new", "main"]
```

`jj pull`一発でfetch+rebase。便利すぎる。

### 実際のワークフロー

俺が「この修正コミットしてプッシュして」って言うと、Claudeがこう動く：

```bash
# 1. 状態確認
jj status

# 2. コミットメッセージ設定
jj describe -m "fix: ブランディングページの表示条件を修正"

# 3. ブックマーク移動（mainブランチ的なやつ）
jj bookmark set main -r @

# 4. プッシュ
jj git push
```

全部Claudeがやってくれる。俺は見てるだけ。最高。

## 84プロジェクトのモノレポ、こうなってる

### なんでモノレポ？

- 共通ライブラリ使い回せる
- 一括検索・置換が楽
- CI/CDも統一できる

### ディレクトリ構成

```
amu-tazawa-scripts/
├── .jj/                    # jujutsu管理
├── kakuduke/               # データ分析系
├── slot_selection/         # スロット選択機能
├── pachi_chatbot/          # チャットボット
├── ai-secretary/           # AI秘書
└── ... (80個以上のプロジェクト)
```

84プロジェクトあっても、jjなら余裕。変更追跡が自動だから、プロジェクト横断の変更もストレスない。

## 【上級者向け】MCP統合

Claude CodeのMCP（Model Context Protocol）使うと、さらに安全にjj操作できる。

```markdown
### MCPツール
| ツール | 何ができる |
|--------|-----------|
| `mcp__jj__status` | 状態確認 |
| `mcp__jj__describe` | コミットメッセージ設定 |
| `mcp__jj__git_push` | プッシュ |
```

構造化された形でjj操作できるから、より安全。

## 困ったときはこれ

### やらかした！ってとき

```bash
# 何でも戻せる。何度でも。
jj undo
jj undo
jj undo

# 操作履歴見たいとき
jj op log
```

**`jj undo`は神。** これがあるから怖くない。

### コンフリクトしても慌てない

jjはコンフリクトを「後回し」にできる。Gitとの決定的な違い。

```bash
# コンフリクトあってもコミットできる
jj describe -m "WIP: コンフリクト解決中"

# 後でゆっくり直す
jj resolve conflicted_file.txt
```

焦らなくていい。後でゆっくり直せばいい。

## まとめ

| 比較 | Git | jujutsu |
|------|-----|---------|
| ステージング | 手動（忘れがち） | 自動（神） |
| 取り消し | 複雑で怖い | `jj undo`で余裕 |
| AI親和性 | ミスりやすい | 安全 |
| 学習コスト | 高い | Git知ってれば楽勝 |

---

**AIがコード書く時代、バージョン管理もAIに任せよう。**

jujutsuは「ミスっても大丈夫」っていう安心感をくれる。Claude Codeと組み合わせたら、バージョン管理の面倒から解放される。

マジで試してみて。世界変わるから。

---

## 参考リンク

- [jujutsu公式](https://github.com/martinvonz/jj)
- [Claude Code](https://claude.ai/claude-code)
- [Conventional Commits](https://www.conventionalcommits.org/)
