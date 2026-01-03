---
title: "Claude Code × jujutsu - AIがバージョン管理を実行する時代"
emoji: "🤖"
type: "tech"
topics: ["jujutsu", "claudecode", "ai", "versioncontrol", "git"]
published: false
---

## はじめに

> 業務自動化Pythonエンジニア。バイブコーディング歴1年 ≒ エンジニア歴。

### gitコマンドすらAIにやらせてる

正直、最初は後ろめたかった。

「gitくらい自分で打てよ」って言われそうで。
`git add` `commit` `push`にやっと慣れたレベルなのに、それすらAIに任せてるなんて。

でも今どう？

**コミットメッセージのAI生成、もはや当たり前じゃない？**

GitHub Copilot、Cursor、みんなやってる。差分見てメッセージ考えるなんてAIの得意分野。だったらコミット操作自体もAIに任せていいはず。

### Claude Codeが来て、状況が変わった

Claude Codeが登場して「これでgit操作も安心」と思った。
git MCPを連携して、AIにバージョン管理を任せる体制が整った。

...はずだった。

**並行開発とMCPの相性、最悪だった。**

`git.lock`が毎回邪魔してくる。複数プロジェクト開いてると必ず詰まる。

簡単な操作はトークンの無駄だから自分で打とうとしてたのに、結局それすら億劫になってしまった。

### エージェント並列開発が当たり前になった

今、私はモノレポで複数プロジェクトを並列開発してる。
`git worktree`じゃなくて、ターミナルで4並列。これが日常。

でもGit + 並列開発 + AI、この組み合わせがどうにも噛み合わない。

### 転機：Claude Code Meetup Tokyo

去年のClaude Code Meetup Tokyoで教えてもらった。

**「エージェント並列開発するなら、jujutsuがいいよ」**

正直、最初は思った。

「え、Git以外のバージョン管理？今さら？」
「Git覚えるのも大変だったのに、また新しいの？」
「てかGitで十分じゃない？」
「新しいツール覚える時間あるなら、コード書きたいんだけど」

わかる。私もそう思ってた。

**大丈夫。jujutsuならね。**

## なぜjujutsuなのか

### Gitの問題、正直キツくない？

正直、普段の`add` `commit` `push`は問題ない。

問題は**たまにしか使わないコマンド**。理解が浅いまま使うから事故る。

私の場合、EC2とローカルMacで同じリポジトリを触ってる。片方で作業した後、もう片方で`git pull`したらコンフリクト。Claudeに「`git stash`しましょう」って言われて実行したら、どこに何がstashされたかわからなくなって詰んだ。

1. **`git stash`の罠** → どこに消えたかわからない
2. **`git reset --hard`の破壊力** → 取り返しつかない
3. **コンフリクト地獄** → 手動で直すしかない

たまにしか使わないから毎回調べる。調べてもよくわからない。AIに任せても、AIも間違える。

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

jj status # 状態確認
jj diff # 差分表示
jj describe -m "msg" # コミットメッセージ設定
jj new # 新しい変更セット作成
jj bookmark set main -r @ # ブックマーク設定
jj git push # リモートにプッシュ
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

## 私のエイリアス設定、公開する

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

私が「この修正コミットしてプッシュして」って言うと、Claudeがこう動く：

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

全部Claudeがやってくれる。私は見てるだけ。最高。

## モノレポでも余裕

複数プロジェクトを1リポジトリで管理するモノレポ構成でも、jjなら問題なし。

- 変更追跡が自動だから、プロジェクト横断の変更もストレスない
- 共通ライブラリ使い回せる
- 一括検索・置換が楽

しかもgitworktreeの代わりの機能もあり

## 実はrulesだけで十分

MCP統合とかスラッシュコマンドとかSkillとか、色々試したけど...

**結論：rulesファイルだけで十分。**

```markdown
# ~/.claude/rules/jj.md に書くだけ

- gitコマンドは使わない → jjを使用
- 主要コマンド一覧
```

これだけでClaude Codeは完璧に理解してくれる。余計な設定いらない。

ちなみにMCP統合は`git.lock`が発生して逆に面倒だった。シンプルが一番。

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

| 比較         | Git              | jujutsu           |
| ------------ | ---------------- | ----------------- |
| ステージング | 手動（忘れがち） | 自動（神）        |
| 取り消し     | 複雑で怖い       | `jj undo`で余裕   |
| AI親和性     | ミスりやすい     | 安全              |
| 学習コスト   | 高い             | Git知ってれば楽勝 |

---

**AIがコード書く時代、バージョン管理もAIに任せよう。**

jujutsuは「ミスっても大丈夫」っていう安心感をくれる。Claude Codeと組み合わせたら、バージョン管理の面倒から解放される。

マジで試してみて。世界変わるから。

---

## 参考リンク

- [jujutsu公式](https://github.com/martinvonz/jj)
- [Claude Code](https://claude.ai/claude-code)
- [Conventional Commits](https://www.conventionalcommits.org/)
