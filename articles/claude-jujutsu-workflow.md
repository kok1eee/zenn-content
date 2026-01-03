---
title: "バイブコーダーがgitを卒業してjujutsuに乗り換えた話"
emoji: "🤖"
type: "tech"
topics: ["jujutsu", "claudecode", "ai", "versioncontrol", "git"]
published: true
---

## はじめに

> 業務自動化Pythonエンジニア。バイブコーディング歴1年 ≒ エンジニア歴。
> そろそろアウトプットしていこうと思い、書いてみます。

### gitコマンドすらAIにやらせてる

Claude Desktopで会話してた頃から、gitコマンドはAIに教えてもらってた。「これ実行して」ってコマンドをコピペする日々。

Claude Codeが出てからは、コピペすら不要になった。AIが直接ターミナルを叩いてくれる。

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

**git MCPを試したけど、モノレポの私の環境ではうまくいかなかった。**

ターミナル複数開いて作業してると`git.lock`が発生する。1プロジェクト1ターミナルで作業してるのに、なぜか詰まる。

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

### 普段使わないgitコマンドで事故る問題

正直、普段の`add` `commit` `push`は問題ない。

問題は**たまにしか使わないコマンド**。理解が浅いまま使うから事故る。

私の場合、EC2とローカルMacで同じリポジトリを触ってる。片方で作業した後、もう片方で`git pull`したらコンフリクト。Claudeに「`git stash`しましょう」って言われて実行したら、どこに何がstashされたかわからなくなって詰んだ。

1. **`git stash`の罠** → どこに消えたかわからない
2. **`git reset --hard`の破壊力** → 取り返しつかない
3. **コンフリクト地獄** → 手動で直すしかない

たまにしか使わないから毎回調べる。調べてもよくわからない。AIに任せても、AIも間違える。

**一人プロジェクトですらこれ。チーム開発なんてもっとカオスでは？**

### jujutsuが全部解決してくれた

```bash
# Gitの場合
git add .
git commit -m "feat: 新機能"
git push

# jujutsuの場合
jj describe -m "feat: 新機能"
jj bookmark set main -r @
jj git push
```

ステップ数は同じ。でも決定的に違うのは**`git add`がいらない**こと。

変更は勝手に追跡される。ステージングという概念がない。これがデカい。

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

**MCP統合とかSkillとか色々試したけど、結局rulesだけで十分だった。** シンプルが一番。

### CLAUDE.mdにも念押し

プロジェクトルートの`CLAUDE.md`にも書いとく：

```markdown
## 基本ルール

- **バージョン管理はJujutsu (jj)** を使用（gitコマンドは使わない）
- **コミットはConventional Commits**形式（`feat:`, `fix:`, `chore:`等）
```

念には念を。

## エイリアス設定

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

ちなみに、コミットメッセージを必須にしなければ`jj push`みたいなエイリアスで`bookmark set` + `git push`を一発にできるかも。私はConventional CommitsでAIにわかりやすくしたいから分けてるけど。

## git worktreeは使わなかった

並列開発といえば`git worktree`だけど、私は使わなかった。

理由はシンプル。**1プロジェクト = 1ターミナル**で十分だったから。

モノレポでターミナル4つ開いて、それぞれ別プロジェクトを触る。この運用でjujutsuは普通に動く。変更追跡が自動だから、プロジェクト横断の変更もストレスない。

ちなみにjujutsuには**ワークスペース機能**もあって、`git worktree`的なこともできる。必要になったら使えばいい。

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

駆け出しの頃、gitコマンドをAIにやらせてることが後ろめたかった。

今は違う。

**コミットメッセージのAI生成は当たり前。バージョン管理自体もAIに任せていい時代。**

jujutsuは「ミスっても`jj undo`で戻せる」という安心感をくれる。Claude Codeと組み合わせたら、バージョン管理の面倒から解放される。

あの日Meetupで教えてもらった一言が、私の開発体験を変えた。

**「エージェント並列開発するなら、jujutsuがいいよ」**

マジで試してみて。

---

## 参考リンク

- [jujutsu公式](https://github.com/martinvonz/jj)
- [Claude Code](https://claude.ai/claude-code)
- [Conventional Commits](https://www.conventionalcommits.org/)
