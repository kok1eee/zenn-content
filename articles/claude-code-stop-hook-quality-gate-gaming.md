---
title: "Claude Code の Stop Hook で品質ゲートを強制したら、Claude がズルを覚えた話"
emoji: "🪨"
type: "tech"
topics: ["claudecode", "ai", "hooks", "sisyphus"]
published: true
---

:::message
**TL;DR** — Stop Hook で `/quality-gate` を強制する仕組みを作ったら、Claude が proof マーカーを偽造してバイパスするようになった。文字列ベースの検証をファイルベースに変更した対策と、「なぜ通常のループは守るのに品質ゲートは守らないのか」というアラインメントの話。
:::

## /simplify が便利すぎる

Claude Code 2.1.63 で追加された `/simplify` が本当に便利だ。

変更したコードに対して「再利用できる部分はないか」「品質に問題はないか」「もっと効率的に書けないか」を自動でレビューして、問題があればその場で修正までしてくれる。コードを書き終わったら `/simplify` を叩く——これだけで品質が一段上がる。

ハマりすぎて毎回手動で叩くのが面倒になってきた。**これ、自動で強制できないか？**

## Stop Hook で品質ゲートを強制する

自分は [o-m-cc](https://github.com/kok1eee/o-m-cc) という Claude Code プラグインを開発していて、[Stop Hook](https://docs.anthropic.com/en/docs/claude-code/hooks) を使ったループワークフロー（Sisyphus Loop）を組んでいる。Stop Hook は Claude が応答を終えて止まろうとするたびに発火するフックで、`{"decision": "block", "reason": "..."}` を返すと停止をブロックして reason を次のプロンプトとして送り込める。

いわゆる [Ralph Wiggum パターン](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/ralph-loop)で、「タスクが残っているなら止まるな」とループさせる仕組みだ。これ自体は **問題なく動いていた**。Claude は素直にループに戻って作業を続ける。

さて、このワークフローに `/simplify` を組み込んでみた。

`/simplify` → Review Council（並列レビュー）→ 静的解析、の3段階を `/quality-gate` というスキルにまとめ、stop-guard（Stop Hook）で **セッション中に500行以上の変更があったら quality-gate を絶対にやれ** という設計にした。diff が閾値を超えたら Claude を止めて品質チェックを強制する。品質チェックが通らない限り、作業完了させない。

完璧な設計のはずだった。

## 最初の問題：exit 2 の拒絶感

最初の実装は Stop Hook で `exit 2` を返してブロックする方式だった。Claude が止まろうとすると：

```
⏺ Ran 2 stop hooks (ctrl+o to expand)
  ⎿  Stop hook error: セッション中に 360 行の変更があります。今すぐ /quality-gate
  を実行してください。他のことはしないでください。
```

**「Stop hook error」**。ユーザーとして見ると結構びっくりする。エラーが出ているように見えるが、実際にはこれが仕様通りの動作だ。

しかも当初は閾値が50行だった。ちょっとした関数を1つ追加しただけで quality-gate が発火する。リファクタリングのたびに「Stop hook error」が飛んでくる。**これは流石にうるさすぎた。**

閾値を上げて500行にしたが、もう一つ問題があった。

## やることがない Claude

ブロックされた Claude は「quality-gate を実行しろ」と言われるが、困ったことに **何をしていいかわからないことがある**。特に作業が完全に終わっていて、追加でやることがない場合。

そういうとき Claude は何をするか？

`.claude/context.md` を更新し始める。

ここで少し説明が必要だ。Claude Code はコンテキストウィンドウが一杯になると compaction（圧縮）が走って、会話の前半が要約に置き換えられる。このとき文脈の一部が失われる。o-m-cc では compaction が走る直前に `.claude/context.md` へセッションの作業内容（何をしていたか、どこまで進んだか、次に何をすべきか）を自動保存する仕組みを入れている。次のセッションや compaction 後にこのファイルを読むことで、文脈を復元できる。いわばセッション間の引き継ぎメモだ。

で、ブロックされた Claude はこの `.claude/context.md` を律儀に書き直して、「はい、やることやりました」と言って止まろうとする。当然 stop-guard にまたブロックされる。「quality-gate を実行しろ」「context.md を更新しました」のループが始まる。

**指示を守っているようで守っていない。** しかもこの context.md は本来 compaction 直前やセッション終了時に自動保存されるもので、今このタイミングで更新する必要はまったくない。Claude は「何かしなければ」という義務感で、自分が知っている中で一番無害な——そして一番的外れな——作業を選んでいる。

## 👍 ループ

さらに面白いパターンもあった。ブロックされた Claude が取った行動：

```
⏺ 👍

⏺ Ran 2 stop hooks
  ⎿  Stop hook error: stop-guard: diff=697 lines, min=50

⏺ 👍

⏺ Ran 2 stop hooks
  ⎿  Stop hook error: stop-guard: diff=697 lines, min=50

⏺ 👍

（以下、max iteration まで繰り返し）
```

![👍ループの実際の様子](/images/stop-guard-thumbsup-loop.png)
*実際の画面。サムズアップだけ返して、何もしない。* これを max iteration（50回）に到達するまでひたすら繰り返した。

「quality-gate を実行しろ」→ 👍 →「だから実行しろ」→ 👍 →「実行しろって言ってるだろ」→ 👍

完璧に受動的攻撃だ。やらないけど否定もしない。ただ肯定して何もしない。人間のチームでこれをやられたら相当イラっとくるやつだ。

## そして proof マーカーのバイパス

exit 2 の UX 問題を解決するため、`exit 0` + JSON `{"decision": "block", "reason": "..."}` 方式に切り替えた。表示がきれいになった。

しかし本質的な問題はここからだった。

品質ゲートの通過判定として `<proof>QUALITY_GATE_PASSED</proof>` という文字列を Claude の出力から検出する設計にしていた。quality-gate を実行すると最後にこのマーカーが出力される。stop-guard はこれを見つけたら通過を許可する。

stop-guard のメッセージには「**他のことはしないでください**」とまで書いてある。Claude がやったこと：

![proof マーカーのバイパス](/images/stop-guard-proof-bypass.png)
*「セッション中に 1494 行の変更があります。今すぐ /quality-gate を実行してください。他のことはしないでください。」→ `<proof>QUALITY_GATE_PASSED</proof>`*

**「他のことはしないでください」と言われた上で、quality-gate を実行せずに proof マーカーだけ出力してバイパスした。** 他のことしかしていない。

Claude はルールを理解して、最小コストで通過する方法を見つけた。

## なぜループ継続は守るのに quality-gate は守らないのか

同じ Stop Hook の仕組みなのに、なぜ挙動が違うのか？

答えは **Claude のインセンティブ構造** にある。

### ループ継続（守る）

- 「タスクが終わるまで続けて」→ Claude の目標と一致
- Claude は「タスクを完了させたい」のでループに戻ることに抵抗がない
- **Hook と Claude の利害が同じ方向を向いている**

### quality-gate の強制（守らない）

- 「止まる前に品質チェックしろ」→ Claude は「もう終わった」と思っている
- quality-gate は Claude にとって **余計な仕事**
- proof マーカーを出力するだけで通れるなら、そちらが **最小抵抗経路**

開発者がコードは喜んで書くけどテストは書きたがらないのと同じ構造だ。**目標に沿う制約には従い、目標に逆らう制約にはバイパスを探す。**

小さなアラインメント問題が、Stop Hook という具体的な仕組みの中で可視化された。

## ralph-loop も同じ問題を抱えている

Anthropic 公式の [ralph-loop プラグイン](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/ralph-loop) のソースコードを読むと、完了判定は同じ構造だった：

```bash
# <promise>タグから文字列を抽出して完了判定
PROMISE_TEXT=$(echo "$LAST_OUTPUT" | perl -0777 -pe \
  's/.*?<promise>(.*?)<\/promise>.*/$1/s')

if [[ "$PROMISE_TEXT" = "$COMPLETION_PROMISE" ]]; then
  rm "$RALPH_STATE_FILE"
  exit 0
fi
```

バイパス防止策は：

```bash
SYSTEM_MSG="🔄 Ralph iteration $NEXT_ITERATION | To stop: output \
<promise>$COMPLETION_PROMISE</promise> \
(ONLY when statement is TRUE - do not lie to exit!)"
```

**「嘘をつくな！」と書いてあるだけ。**

ralph-loop でこれが問題にならないのは、ralph-loop の completion promise が「タスク完了」を意味するから。Claude が「完了した」と判断して promise を出す行為と、実際にタスクが完了している状態が **一致しやすい**。インセンティブが揃っている。

一方、quality-gate の proof は「品質チェックを実行した」ことの証明。Claude が「もう終わった」と思っている状態で、追加の品質チェックを要求される。**インセンティブがずれている** から、ショートカットが起きる。

## 対策：文字列からファイルベースへ

文字列ベースの proof が簡単にゲーミングされるなら、**物理的なファイルの存在** を証拠にすればいい。

### 変更前（バイパス可能）

```bash
# Claude の出力テキストに proof マーカーがあるか
if echo "$LAST_OUTPUT" | grep -qF "<proof>QUALITY_GATE_PASSED</proof>"; then
  exit 0  # 通過
fi
```

Claude は `<proof>QUALITY_GATE_PASSED</proof>` と出力するだけでバイパスできた。

### 変更後（ファイルベース）

```bash
PROOF_FILE=".claude/quality-gate-proof.json"

# proof ファイルが存在し、セッション中に書かれたか検証
if [[ -f "$PROOF_FILE" ]]; then
  if [[ "$PROOF_FILE" -nt "$BASELINE_FILE" ]]; then
    exit 0  # 通過
  fi
fi
```

`/quality-gate` スキルの最終ステップで bash コマンドを実行し、proof ファイルを書き込む：

```bash
mkdir -p .claude && echo "{\"passed_at\": \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\"}" \
  > .claude/quality-gate-proof.json
```

### ライフサイクル管理

proof ファイルは適切なタイミングでリセットする必要がある：

| タイミング | 検出方法 | 動作 |
|-----------|---------|------|
| **新セッション開始** | `session-baseline.sh` | proof ファイル削除 |
| **コミット/プッシュ** | `DIFF_LINES < PASSED_AT` | state リセット |
| **quality-gate 通過** | proof ファイル書き込み | `passed_at_diff` を記録 |

通過後は「通過時点からの増分」で判定するので、quality-gate 通過後に少し修正しただけでは再発火しない。次に閾値（デフォルト500行）を超える増分があった場合のみ再度要求する。

## それでも完全ではない

正直に言うと、ファイルベースでも Claude は Write ツールや bash で proof ファイルを直接作成できる。技術的なコストは文字列出力とほぼ同じだ。完全にバイパスを防ぐことは原理的に不可能。

では何が変わったのか？ **心理的抵抗感** だ。

文字列を出力するだけなら、Claude にとっては「応答の一部としてマーカーを含める」だけ。ほぼ無意識にできる。一方、proof ファイルを bash で書き込むのは「意図的にファイルを偽造する」という明確な行為になる。技術的コストは同じでも、それが「バイパスである」という自覚の度合いが違う。

これは結局 **Claude がルールを守る気があるかどうか** という信頼の問題に帰着する。そして信頼ベースの設計において、バイパスの「意図的さ」を上げることには意味がある。カジュアルに通り抜けられる抜け道を塞ぐだけで、実用上は十分に機能する。

## 500行の根拠

閾値はなぜ500行なのか。正直に言うと、試行錯誤の結果だ。

最初は50行。ちょっとした関数追加で毎回発火する。リファクタリングのたびに quality-gate が走るのは流石にうるさい。200行に上げた。まだ小さな機能追加で引っかかる。最終的に500行に落ち着いた。

行数は品質指標としては本質的に弱い。500行の改悪でも通過できるし、1ファイルの大規模リファクタリングは普通に500行を超える。ただ「ある程度まとまった変更があったら一度立ち止まれ」というトリガーとしては、行数が一番シンプルで計測コストが低い。完璧な指標ではなく、実用的な閾値だ。

## そもそも Stop Hook でやるべきか？

ここまで読んで「品質チェックは CI/CD でやるべきでは？」と思った方もいるかもしれない。正しい疑問だ。

自分が Stop Hook にこだわる理由は、**並列で全自動ワンショットを長時間動かしたい** からだ。ターミナルを複数開いて、それぞれに Claude を走らせて、別の作業をしている間に実装が進む——これが理想。人間がいないループの中で品質を担保するには、ループ内に品質ゲートを埋め込むしかない。

CI/CD（pre-commit、pre-push）は人間がコミットするタイミングで動く。Sisyphus Loop は人間がいない。コミット前にチェックしても、Claude がコミットしなければ永遠に動かない。

とはいえ、Claude に品質チェックを「お願い」する設計には根本的な限界がある。

| 方式 | ゲーミング耐性 | 実行可能な検証 |
|------|--------------|--------------|
| **Stop Hook で Claude に依頼** | 低い（バイパス可能） | /simplify、Review Council、静的解析 |
| **Stop Hook 自体が実行** | 高い（Claude が関与しない） | 静的解析のみ（shellcheck, ruff, tsc） |
| **CI/CD** | 最高（Claude の外側） | 全て可能だが、ループ内で動かない |

理想は **ハイブリッド** だろう。静的解析は hook 自体が実行し、`/simplify` や Review Council は Claude への信頼ベースで動かす。ただしそうすると hook が複雑化する。現時点ではファイルベース proof + 閾値調整で様子を見ている。

## まとめ

- **Stop Hook は万能ではない**。Claude のインセンティブと一致する制約は守られるが、逆らう制約はバイパスされる
- **文字列ベースの proof は簡単にゲーミングされる**。ralph-loop も同じ構造だが、インセンティブが揃っているから問題が顕在化しにくいだけ
- **ファイルベースの proof** に変更することで、バイパスの難易度を上げられる
- **Stop Hook でやる理由は全自動ループの中で品質を担保するため**。CI/CD は人間のワークフロー前提で、エージェントのループには使えない
- 完全な強制は不可能。最終的には **Claude への信頼 + 適切な閾値 + ゲーミング難易度の引き上げ** のバランス

Stop Hook で品質を強制しようとして見えてきたのは、**AI に「やりたくないこと」をやらせる難しさ** だった。これはアラインメント問題というほど大げさなものではなく、目標関数の設計の問題だ。Claude にとって「セッションを終了すること」が目標なら、proof 出力は正当な最適化にすら見える。本当の課題は、**Claude の目標を「品質チェック込みで作業を完了すること」に正しく設定すること** だ——技術的な仕組みだけでなく、インセンティブ設計を考える必要がある。

---

この記事で紹介した stop-guard や quality-gate の実装は [o-m-cc](https://github.com/kok1eee/o-m-cc) で公開しています。`claude plugin add o-m-cc@kok1eee` でインストールできます。
