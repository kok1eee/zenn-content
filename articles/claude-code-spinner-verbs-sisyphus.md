---
title: "Claude Code 2.1.23 でスピナーの表示が変更できるようになった"
emoji: "🪨"
type: "tech"
topics: ["claudecode", "ai", "cli", "tips"]
published: true
---

## はじめに

> 業務自動化Pythonエンジニア。バイブコーディング歴1年 ≒ エンジニア歴。

Claude Code 2.1.23 で `spinnerVerbs` という設定が追加された。処理中に表示されるスピナーのテキストをカスタマイズできる。

```
// デフォルト
⏳ Thinking...
⏳ Working...
⏳ Processing...

// カスタム
⏳ 岩を押し上げています...
⏳ 神々に抗っています...
```

## 設定方法

`~/.claude/settings.json` に追加する。

```json
{
  "spinnerVerbs": {
    "mode": "replace",
    "verbs": ["表示テキスト1", "表示テキスト2", "表示テキスト3"]
  }
}
```

### mode の違い

| mode | 説明 |
|------|------|
| `"replace"` | デフォルトを完全に置き換え |
| `"append"` | デフォルトに追加（既存 + カスタム） |

### ハマりポイント：配列ではなくオブジェクト

直感的に配列で書きたくなるが、エラーになる。

```json
// ❌ エラー
{
  "spinnerVerbs": ["Thinking", "Working"]
}
```

```
spinnerVerbs: Expected object, but received array
Files with errors are skipped entirely, not just the invalid settings.
```

正しくはオブジェクトで `mode` と `verbs` を指定する。

```json
// ✅ 正しい形式
{
  "spinnerVerbs": {
    "mode": "replace",
    "verbs": ["Thinking", "Working"]
  }
}
```

## 公式ドキュメントにはまだない

2.1.23 の Changelog には `Added customizable spinner verbs setting (spinnerVerbs)` と書いてあるが、設定ページには記載がない。

[GitHub issue #21599](https://github.com/anthropics/claude-code/issues/21599) でもドキュメント不足が報告されている。

### フォーマットの見つけ方

バイナリを直接解析して見つけた。

```bash
strings $(which claude) | grep "spinnerVerb"
```

```
spinnerVerbs: Customize spinner verbs ({ "mode": "append" | "replace", "verbs": [...] })
```

## o-m-cc で実際にやってみた

自分が作っている Claude Code 用のオーケストレータープラグイン [o-m-cc](https://github.com/kok1eee/o-m-cc) で実際に設定してみた。

o-m-cc は「Sisyphus Loop」という思想で動いている。ギリシャ神話のシーシュポス — 永遠に岩を山頂に押し上げ続ける男。タスクが完了するまで止まらない。

スピナーもこの世界観に合わせた。

```json
{
  "spinnerVerbs": {
    "mode": "replace",
    "verbs": [
      "岩を押し上げています",
      "山頂を目指しています",
      "また麓から登っています",
      "永遠に繰り返しています",
      "岩を転がしています",
      "頂上まで押し上げています",
      "神々に抗っています",
      "不屈の意志で格闘しています",
      "岩と格闘しています",
      "運命に立ち向かっています"
    ]
  }
}
```

処理中の表示：

```
⏳ 岩を押し上げています...
⏳ 山頂を目指しています...
⏳ また麓から登っています...
⏳ 神々に抗っています...
```

Claude が考えている間もシーシュポスが岩を押し上げている。かわいい。

### /init で自動設定

o-m-cc の `/o-m-cc:init` コマンドにも組み込んだ。プロジェクト初期化時に「Sisyphus スピナーを設定しますか？」と聞いて、選択すると `~/.claude/settings.json` に自動追加される。既に設定済みなら上書きしない。

## まとめ

- Claude Code 2.1.23 で `spinnerVerbs` が追加された
- フォーマットは `{ "mode": "replace" | "append", "verbs": [...] }`
- 配列ではなくオブジェクト（ハマりポイント）
- 公式ドキュメントはまだない（バイナリ解析で発見）
- 日本語も使える

GitHub: [kok1eee/o-m-cc](https://github.com/kok1eee/o-m-cc)

## 参考

- [Claude Code Changelog 2.1.23](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md)
- [GitHub issue #21599 - spinnerVerbs ドキュメント不足](https://github.com/anthropics/claude-code/issues/21599)
