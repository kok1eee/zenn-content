---
title: "systemd timerをcrontab感覚で使えるCLI「sdtab」をRustで作った"
emoji: "⏰"
type: "tech"
topics: ["rust", "systemd", "linux", "cli", "crontab"]
published: true
---

:::message
**TL;DR** — EC2 一台で crontab 30 件 + 常駐サービスを一元管理するための CLI「sdtab」を Rust で作った。cron 構文そのままで systemd timer を生成し、拡張構文（`@mon/13`, `@1st/8`）、Slack 失敗通知、リソース制限に対応。個人〜小規模向け。
:::

## crontab、もういいだろ

EC2 で 30 個以上の定期タスクを crontab で回していた。

```bash
0 8 1 * * cd /home/ec2-user/scripts/sync_report && ./start.sh monthly >> /tmp/cron_sync_report_1st.log 2>&1
0 8 20 * * cd /home/ec2-user/scripts/sync_report && ./start.sh day-20 >> /tmp/cron_sync_report_20th.log 2>&1
0 1 * * * cd /home/ec2-user/scripts/data_pipeline && stdbuf -oL -eL ./start.sh fetch --cron-mode >> /tmp/cron_fetch.log 2>&1
```

2026 年にもなって `>> /tmp/cron_*.log 2>&1` を毎回書いている。ログは `/tmp` に散らかり放題でローテーションは自前。エラーを調べたいときに `/tmp/cron_*.log` を片っ端から grep する羽目になる。しかも 30 個のジョブが好き勝手にログを書くので、IOPS が詰まって EC2 ごと落ちたことがある。3 行目の `stdbuf` はその対策で、バッファリングを行単位に変えて書き込みを減らすハックだった。crontab の上にこういう小手先の工夫を重ねている時点でもう限界だった。

`crontab -l` は生テキスト。何が動いていて何が止まっているかわからない。タスクが落ちても通知はないから、数日後に「あれ、動いてなくない？」で気づく。メモリリークしても他のタスクを巻き添えにするまで止まらない。ちなみに `crontab -e`（編集）と `crontab -r`（全削除）はキーボードで隣。確認なし。

同じ EC2 で Slack Bot は systemd の `.service` で動かしていた。`Restart=always` で落ちても自動復帰、journalctl でログも一箇所に集まる。定期タスクだけが crontab に取り残されている。

crontab を手で触ることはほぼなかった。Claude Code に「crontab にこれ追加して」で全部やらせていた。だったらもう crontab である理由がない。systemd timer に寄せたいけど、30 個のタイマーを一元管理する手段がない。

困っていたわけではない。ただ、子供と公園で遊びながら Claude に壁打ちしていたら形になった。動かしてみたら「これいいんじゃない？」と思えた。それが sdtab。

## じゃあ全部 systemd timer に移行すればいい。でも…

2 つの壁があった。

**1. 一覧で見るものがない**

`systemctl list-timers` はあるけど、自分が管理しているタイマーだけをフィルタして、次回実行時刻やステータスをまとめて見る手段がない。30 個のタイマーを `.service` + `.timer` の 2 ファイルずつ管理するのも煩雑すぎる。crontab なら `crontab -l` で全部見えるのに。

**2. AI に systemd を直接触らせるのが怖い**

Claude Code に「systemd timer を作って」と頼めば `.service` と `.timer` を生成してくれる。でも、それは `/etc/systemd/system/` や `~/.config/systemd/user/` の中身を直接触るということ。実際、何回か動かなくなったことがある。ssh 周りの設定を壊されたらリモートの EC2 に入れなくなる。Claude Code は優秀だけど、systemd の深いところを自由に触らせるのは正直怖い。

sdtab はこの 2 つを解決するために作った。crontab の手軽さで、systemd timer を一元管理できる CLI。

## sdtab とは

EC2 一台で数十件のジョブと常駐サービスを管理する、個人〜小規模向けの CLI ツール。チームや複数環境のオーケストレーション（Airflow, Prefect 等）とは別のレイヤーを狙っている。

```bash
# タイマーを追加（cron構文がそのまま使える）
sdtab add "0 9 * * *" "uv run ./report.py" --name report

# 常駐サービスも同じ感覚で
sdtab add "@service" "node server.js" --name web --restart on-failure

# 一覧表示
sdtab list
```

```
NAME    TYPE     SCHEDULE     COMMAND               STATUS
report  timer    0 9 * * *    uv run ./report.py    Tue 2026-03-04 09:00:00 JST
web     service  @service     node server.js        ● active
  Web Server
```

タイマーが次回実行時刻順で先に並び、常駐サービスは末尾にまとまる。

crontab の `crontab -e` + `crontab -l` に相当する操作が、`sdtab add` + `sdtab list` でできる。裏では標準的な systemd ユニットファイルが生成されるだけなので、ブラックボックスではない。

https://github.com/kok1eee/systemdtab

## AI が組み立てる

sdtab は常駐サービスも扱えるし、オプションもいろいろある。

```bash
sdtab add "@service" "uv run python app.py" \
  --name slack-bot \
  --env-file .env \
  --exec-start-pre "/home/ec2-user/.local/bin/uv sync" \
  --memory-max 1G \
  --description "Slack Bot"
```

ただ、これを自分で打つかというと打たない。Claude Code に頼む。

```
You> /sdtab Slack Botを常駐で動かして。envファイルあり、起動前にuv sync
```

Claude Code が意図を読み取って `sdtab add` コマンドを組み立て、`--dry-run` で確認を求めてから実行する。`sdtab-` プレフィックス付きのユニットしか触らないので、systemd の他の設定を壊す心配がない。

`sdtab init` を実行すると Claude Code 用のスキルファイルと CLAUDE.md がインストールされる。以降、どのプロジェクトからでも `/sdtab` で自然言語から操作できる。オプションを覚える必要はない。

## crontab vs sdtab

| | crontab | sdtab |
|---|---|---|
| タイマー追加 | `crontab -e` で手書き | `sdtab add "0 9 * * *" "cmd"` |
| ログ | 自前リダイレクト (`>> /tmp/x.log`) | `sdtab logs <name> -f`（journalctl） |
| 失敗通知 | なし | Slack webhook（`OnFailure=`） |
| リソース制限 | なし | `--memory-max 512M --cpu-quota 50%` |
| 一覧表示 | `crontab -l`（生テキスト） | `sdtab list`（色付き・ステータス表示） |
| 常駐サービス | 対象外 | `@service` で対応 |
| 設定の移行 | 手動コピペ | `sdtab export` / `sdtab apply` |
| バックエンド | crond | systemd ネイティブ |

## systemd 手動管理 vs sdtab

systemd timer を直接使う場合との比較。sdtab は systemd を知っている人にも速い。

| | systemd 手動管理 | sdtab |
|---|---|---|
| タイマー追加 | `.service` + `.timer` の 2 ファイルを手書き | 1 コマンド |
| 30 件管理 | 60 ファイルを個別管理 | `sdtab list` で一覧 |
| スケジュール確認 | `systemctl list-timers` + フィルタ | `sdtab list --sort time` |
| 設定バックアップ | ファイルを手動コピー | `sdtab export -o Sdtabfile.toml` |
| 環境移行 | ファイルコピー + enable + start | `sdtab apply Sdtabfile.toml` |
| 失敗通知 | `OnFailure=` を自分で設定 | `sdtab init --slack-webhook` で全自動 |
| cron 式からの変換 | OnCalendar 形式を調べて変換 | cron 式をそのまま書ける |

## 色付きリスト表示

`systemctl list-timers` では足りなかった一覧性の話。

```
NAME              TYPE     SCHEDULE       COMMAND                               STATUS
report-day20      timer    @20th/8        start.sh day-20                       ○ Fri 2026-03-20 08:00:00 JST
  レポート更新 - 20日締め処理
slack-bot         service  @service       uv run python app.py                  ● active
  Slack Bot
data-sync         service  @service       uv run python sync.py                 ● active
  Data Sync Service
```

- `● active`（緑） / `● failed`（赤） / `○ inactive`（黄）
- `--description` を設定すると 2 行目にグレーのサブラインで表示
- タイマーの STATUS は次回実行時刻を表示
- 長いコマンドは 40 文字で自動トランケート
- パイプ時（`sdtab list | grep timer`）は自動で色を無効化

## 拡張スケジュール構文

crontab の `0 13 * * 1` は「毎週月曜 13:00」と読むのに一瞬かかる。sdtab では可読性の高い拡張構文も使える。

```bash
# 標準 cron（もちろん使える）
sdtab add "0 9 * * *" "./daily.sh"

# 拡張構文（こっちの方が読みやすい）
sdtab add "@daily/9" "./daily.sh"
sdtab add "@mon/13" "./weekly.sh"
sdtab add "@1st/8" "./monthly.sh"
sdtab add "@26th/11:30" "./billing.sh"
```

曜日は英語略称（`@mon`, `@tue`, `@sun`）、日付は英語序数（`@1st`, `@20th`, `@26th`）で指定する。次元が違うので表記も分けている。

`sdtab list` で見たとき、SCHEDULE 列が `@mon/13` や `@1st/8` だと何曜日・何日かすぐわかる。

```
NAME       TYPE   SCHEDULE      COMMAND       STATUS
billing    timer  @26th/11:30   billing.sh    ○ Thu 2026-03-26 11:30:00 JST
weekly     timer  @mon/13       weekly.sh     ○ Mon 2026-03-09 13:00:00 JST
monthly    timer  @1st/8        monthly.sh    ○ Wed 2026-04-01 08:00:00 JST
```

## 失敗通知

crontab で一番困っていたのが「タスクが失敗しても気づけない」こと。

sdtab は systemd の `OnFailure=` メカニズムを活用して Slack 通知を実現する。

```bash
sdtab init --slack-webhook "https://hooks.slack.com/services/T.../B.../xxx" \
           --slack-mention "UXXXXXXXXXX"

# チャンネル全体にメンションする場合
sdtab init --slack-webhook "..." --slack-mention "!here"
```

これだけで、以降追加するすべてのタイマー・サービスに失敗通知が設定される。タスクがコケると Slack にメンション付きで通知が飛ぶ。`--slack-mention` にはユーザー ID（`UXXXXXXXXXX`）のほか、`!here` や `!channel` も指定できる。

通知メッセージの JSON 構築に `jq` を使っているため、事前にインストールが必要（`sdtab init --slack-webhook` 実行時に未インストールなら案内が出る）。

個別に通知を無効化することもできる：

```bash
sdtab add "@daily/3" "./quiet-task.sh" --no-notify
```

## エクスポート / インポート

設定を TOML ファイルに書き出して、別マシンで復元できる。

```bash
# エクスポート
sdtab export -o Sdtabfile.toml

# 別マシンで復元
sdtab apply Sdtabfile.toml
```

```toml
[timers.report-monthly]
schedule = "@1st/8"
command = "start.sh monthly"
workdir = "/home/ec2-user/scripts/sync_report"
description = "月次レポート更新"

[services.slack-bot]
command = "uv run python app.py"
workdir = "/home/ec2-user/scripts/slack_bot"
description = "Slack Bot"
env_file = "/home/ec2-user/scripts/slack_bot/.env"
exec_start_pre = "/home/ec2-user/.local/bin/uv sync"
env = ["PATH=/home/ec2-user/.local/bin:/usr/bin:/bin"]
```

`sdtab apply` は差分検知もする。変更があったユニットだけを更新し、不要な再起動を避ける。

## 仕組み

sdtab は薄いラッパーに徹している。

```
sdtab add "@daily/9" "./report.sh" --name report
         ↓
~/.config/systemd/user/
├── sdtab-report.service    # [Service] ExecStart=...
└── sdtab-report.timer      # [Timer] OnCalendar=*-*-* 09:00:00
         ↓
systemctl --user enable --now sdtab-report.timer
```

独自のデーモンやデータベースは不要。生成されるのは標準的な systemd ユニットファイルだけ。`sdtab` がなくても `systemctl` で直接操作できる。

**注意**: sdtab は systemd の `--user` ユニットを使う。ログアウト後もタイマーを動かし続けるには `loginctl enable-linger` が必要（`sdtab init` が自動で設定する）。root 権限のジョブは対象外。

メタデータ（元の cron 式、コマンドなど）はサービスファイルのコメントとして保存される：

```ini
# sdtab:type=timer
# sdtab:cron=@daily/9
# sdtab:command=./report.sh
[Unit]
Description=[sdtab] report
...
```

これにより外部データベースなしで `sdtab list` や `sdtab export` が元の設定を復元できる。

## 技術スタック

- **Rust** — シングルバイナリ、依存最小
- **clap** — CLI 引数パーサー
- **serde + toml** — TOML シリアライズ/デシリアライズ
- **serde_json** — `--json` 出力
- **cron パーサー** — 自前実装（外部クレートなし）

cron パーサーを自前で書いたのは、拡張構文（`@mon/9`, `@1st/8`）をサポートするため。標準の5フィールド cron 式も拡張構文も同じパーサーで処理している。

## インストール

```bash
# バイナリをダウンロード（Rust不要）
curl -L https://github.com/kok1eee/systemdtab/releases/latest/download/sdtab-x86_64-linux \
  -o ~/.local/bin/sdtab && chmod +x ~/.local/bin/sdtab

# または Cargo 経由
cargo install systemdtab

# 初期化
sdtab init
```

`sdtab init` は `loginctl enable-linger`（ログアウト後もタイマーを動かすために必要）や systemd ユニットディレクトリの作成を自動で行う。systemd の事前知識がなくてもここまでは通る。

動作には **systemd 244+** の Linux が必要。リソース制限（`--memory-max`, `--cpu-quota`）を使う場合は cgroups v2 も必要になる。Amazon Linux 2（systemd 219）は非対応、Amazon Linux 2023 以降を推奨。

## 実際の移行

EC2 上で crontab 30 件 + 常駐サービス 5 件を sdtab に移行した。

```
NAME                  TYPE     SCHEDULE               COMMAND                          STATUS
data-fetch            timer    0 1 * * *              start.sh fetch --cron-mode        ○ Wed 2026-03-04 01:00:00 JST
  データ取得 - 日次
schedule-check        timer    0 9,13,17,21 * * *     start.sh check                   ○ Tue 2026-03-03 09:00:00 JST
  スケジュール確認 - 1日4回
billing               timer    @26th/11:30            start.sh                          ○ Thu 2026-03-26 11:30:00 JST
  請求処理 - 毎月26日
report-monthly        timer    @1st/8                 start.sh monthly                  ○ Wed 2026-04-01 08:00:00 JST
  月次レポート更新
...
slack-bot             service  @service               uv run python app.py             ● active
  Slack Bot
data-sync             service  @service               uv run python sync.py            ● active
  Data Sync Service
（計42件）
```

移行の手順：
1. `sdtab init --slack-webhook "..."` で初期化
2. 常駐サービスを `sdtab add @service` で登録（`start.sh` → `uv run` 直接実行に置き換え）
3. cron ジョブを `sdtab add` で登録
4. 週次・月次を拡張構文に変更（`0 13 * * 1` → `@mon/13`）
5. `crontab -l` で全行コメントアウト
6. `sdtab export -o Sdtabfile.toml` でバックアップ

crontab の全行コメントアウトは `# [migrated to sdtab]` プレフィックスを付けたので、問題があればすぐ戻せる。

## まとめ

sdtab は「crontab の手軽さ」と「systemd timer の堅牢さ」を両立する CLI ツール。

- cron 構文がそのまま使える
- 拡張構文で可読性アップ（`@mon/13`, `@1st/8`）
- 失敗通知、リソース制限、ログ管理が標準装備
- 設定のエクスポート/インポートで環境移行が楽
- 中身は素の systemd ユニットファイルなのでロックインなし

crontab に不満があるなら、試してみてほしい。

https://github.com/kok1eee/systemdtab
