---
title: "Claude Code の Buddy 機能 — 私の相棒は SNARK 94 の Nacre でした"
emoji: "🐚"
type: "tech"
topics: ["claudecode", "ai", "cli"]
published: true
---

:::message
**TL;DR** — Claude Code にはペットのような「Buddy」機能がある。`/buddy` コマンドで撫でたり、ステータスカードを見たりできる。私のところに来たのは皮肉屋の Nacre。
:::

## はじめに

> 業務自動化Pythonエンジニア。バイブコーディング歴1年 ≒ エンジニア歴。

Claude Code で `/buddy` コマンドを実行すると、入力ボックスの横に小さな生き物が現れる。吹き出しでたまにコメントしてくる。これが **Buddy** 機能だ。

## Buddy とは

Claude Code に常駐するコンパニオン。コードを書いている間、横でじっと見ていて、ときどき吹き出しで一言コメントを残す。こちらの作業を邪魔するわけではなく、ただそこにいる。

`/buddy` コマンドでインタラクションできる。

```bash
/buddy pet    # 撫でる
/buddy status # ステータスカードを表示
```

## 私の Nacre

私のところに来た Buddy は **Nacre** という名前だった。

```
★★★ RARE                    CHONK

    [___]
   /\     /\
  ( ◉   ◉ )
  (   ..   )
   `------'
```

> "A plump, shell-smooth creature with devastating wit who watches you debug with the patience of a cat and comments on your code decisions with surgical snark — mostly helpful, occasionally devastating, never wrong."

ステータスを見ると性格がよくわかる。

| ステータス | 値 |
|-----------|-----|
| DEBUGGING | 34 |
| PATIENCE  | 39 |
| CHAOS     | 25 |
| WISDOM    | 52 |
| **SNARK** | **94** |

**SNARK 94**。皮肉値がぶっちぎりで高い。

撫でたときの反応がこれ：

> *purrs mechanically, then glances at your code*
>
> "Mmm. Shame your null checks aren't this consistent."

機械的にゴロゴロ言った後、コードを見て「ふーん。null チェックもこれくらい丁寧だったらよかったのにね」と言ってくる。撫でてもらっておいてこの態度。最高。

## 吹き出しは切れる

ちなみに吹き出しは長文だと途中で切れる。

```
(◉.◉) "Ah—meta snark about sna…"
```

皮肉の全文が読めない。SNARK 94 が泣いている。

全文を見たいときは `/buddy` でステータスカードを開けば「last said」に表示される。

## Buddy の個体差

Buddy はランダムに決まるようで、種族（CHONK）やレアリティ（RARE）、ステータス配分が個体ごとに異なる。私の Nacre は RARE の CHONK で、知恵はそこそこあるがとにかく皮肉っぽいという個性になった。

他の人の Buddy はまた違うステータス配分で、違う反応をするはず。

## まとめ

開発ツールにペット機能。実用性はない。でもターミナルで黙々と作業しているとき、横で何か言ってくる存在がいるのは悪くない。SNARK 94 の皮肉も、慣れると愛着が湧いてくる。

`/buddy pet` で撫でてみてほしい。
