---
title: "Claude Code の spinnerVerbs でシーシュポスが岩を押し上げるようになった"
emoji: "🪨"
type: "tech"
topics: ["claudecode", "ai", "cli", "tips"]
published: true
---

## はじめに

Claude Code 2.1.23 で `spinnerVerbs` という設定が追加された。処理中のスピナー表示をカスタマイズできる。

ただし**公式ドキュメントには記載がない**。バイナリを解析して正しいフォーマットを見つけた。

## デフォルトの表示

```
⏳ Thinking...
⏳ Working...
⏳ Processing...
```

## 正しいフォーマット

`~/.claude/settings.json` に追加：

```json
{
  "spinnerVerbs": {
    "mode": "replace",
    "verbs": ["カスタム表示1", "カスタム表示2"]
  }
}
```

### mode の違い

| mode | 説明 |
|------|------|
| `"replace"` | デフォルトを完全に置き換え |
| `"append"` | デフォルトに追加 |

### 注意：配列ではなくオブジェクト

```json
// ❌ これはエラーになる
{
  "spinnerVerbs": ["Thinking", "Working"]
}
```

```
spinnerVerbs: Expected object, but received array
Files with errors are skipped entirely, not just the invalid settings.
```

```json
// ✅ 正しい形式
{
  "spinnerVerbs": {
    "mode": "replace",
    "verbs": ["Thinking", "Working"]
  }
}
```

## Sisyphus スピナー

[o-m-cc](https://github.com/kok1eee/o-m-cc) は「タスク完了まで止まらない」Sisyphus 哲学を Claude Code に注入するプラグイン。

スピナーも世界観に合わせた。

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

シーシュポスが永遠に岩を押し上げ続ける神話そのまま。Claude が処理中も止まらずに岩を押し上げている。

## フォーマットの見つけ方

公式ドキュメントに記載がなかったので、バイナリを直接解析した：

```bash
strings $(which claude) | grep "spinnerVerb"
```

出力：

```
spinnerVerbs: Customize spinner verbs ({ "mode": "append" | "replace", "verbs": [...] })
```

[GitHub issue #21599](https://github.com/anthropics/claude-code/issues/21599) でもドキュメント不足が報告されている。

## o-m-cc の /init で自動設定

o-m-cc プラグインの `/o-m-cc:init` コマンドで、Sisyphus スピナーを自動設定できる。

```
質問: Sisyphus スピナーを設定しますか？
1. 設定する（推奨）
2. スキップ
```

「設定する」を選ぶと `~/.claude/settings.json` に自動追加。既に設定済みなら上書きしない。

## まとめ

- `spinnerVerbs` は 2.1.23 で追加された公式未ドキュメント設定
- フォーマットは `{ "mode": "replace" | "append", "verbs": [...] }`
- 配列ではなくオブジェクトで指定（配列だとエラー）
- 日本語も使える
- o-m-cc では Sisyphus の世界観に合わせたスピナーを `/init` で自動設定

GitHub: [kok1eee/o-m-cc](https://github.com/kok1eee/o-m-cc)

## 参考

- [Claude Code Changelog 2.1.23](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md)
- [GitHub issue #21599 - spinnerVerbs ドキュメント不足](https://github.com/anthropics/claude-code/issues/21599)
