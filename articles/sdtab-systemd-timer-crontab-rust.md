---
title: "systemd timerをcrontab感覚で使えるCLI「sdtab」をRustで作った"
emoji: "⏰"
type: "tech"
topics: ["rust", "systemd", "linux", "cli", "crontab"]
published: false
---

## crontab、つらくないですか？

EC2 で 30 個以上の定期タスクを crontab で管理していた。

```bash
0 8 1 * * cd /home/ec2-user/scripts/update_salesdata && ./start.sh monthly-start >> /tmp/cron_update_salesdata_1st.log 2>&1
0 8 20 * * cd /home/ec2-user/scripts/update_salesdata && ./start.sh day-20 >> /tmp/cron_update_salesdata_20th.log 2>&1
0 1 * * * cd /home/ec2-user/scripts/media_manager && stdbuf -oL -eL ./start.sh plusmade --cron-mode >> /tmp/cron_plusmade.log 2>&1
```

つらいポイント：

- **ログが散らばる** — `/tmp/cron_*.log` が大量に生まれる。ローテーションは自前
- **失敗に気づけない** — コケても通知がない。気づいたら数日止まってた
- **一覧性がない** — `crontab -l` は生テキスト。何が動いていて何が止まっているかわからない
- **リソース制限ができない** — メモリリークで他のタスクを巻き込む
- **環境ごとの再構築が面倒** — 新しいサーバーに `crontab -e` で手打ち

systemd timer はこれらを全部解決できる。でも、ユニットファイルを2つ（`.service` + `.timer`）書いて `systemctl enable --now` するのが面倒すぎる。

**crontab の手軽さで systemd timer の恩恵を受けたい。** それで作ったのが sdtab。

## sdtab とは

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

crontab の `crontab -e` + `crontab -l` に相当する操作が、`sdtab add` + `sdtab list` でできる。裏では標準的な systemd ユニットファイルが生成されるだけなので、ブラックボックスではない。

https://github.com/kok1eee/systemdtab

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
           --slack-mention "U0700J8MN3W"
```

これだけで、以降追加するすべてのタイマー・サービスに失敗通知が設定される。タスクがコケると Slack にメンション付きで通知が飛ぶ。

個別に通知を無効化することもできる：

```bash
sdtab add "@daily/3" "./quiet-task.sh" --no-notify
```

## 色付きリスト表示

```
NAME                TYPE     SCHEDULE       COMMAND                               STATUS
ambient-task-agent  service  @service       ambient-task-agent serve --port 3100  ● active
batch-download      service  @service       uv run python bolt_batch_download.py  ● active
  Batch Download Service - Kintone to Excel via Slack Bot
salesdata-day20     timer    @20th/8        start.sh day-20                       ○ Fri 2026-03-20 08:00:00 JST
  売上データ更新 - 20日締め処理
```

- `● active`（緑） / `● failed`（赤） / `○ inactive`（黄）
- `--description` を設定すると 2 行目にグレーのサブラインで表示
- タイマーの STATUS は次回実行時刻を表示
- 長いコマンドは 40 文字で自動トランケート
- パイプ時（`sdtab list | grep timer`）は自動で色を無効化

## エクスポート / インポート

設定を TOML ファイルに書き出して、別マシンで復元できる。

```bash
# エクスポート
sdtab export -o Sdtabfile.toml

# 別マシンで復元
sdtab apply Sdtabfile.toml
```

```toml
[timers.salesdata-monthly-start]
schedule = "@1st/8"
command = "start.sh monthly-start"
workdir = "/home/ec2-user/scripts/update_salesdata"
description = "売上データ更新 - 月初処理"

[services.slack-task-runner]
command = "uv run python app.py"
workdir = "/home/ec2-user/scripts/slack_task_runner"
description = "Slack Task Runner"
env_file = "/home/ec2-user/scripts/slack_task_runner/.env"
exec_start_pre = "/home/ec2-user/.local/bin/uv sync"
env = ["PATH=/home/ec2-user/.local/bin:/usr/bin:/bin"]
```

`sdtab apply` は差分検知もする。変更があったユニットだけを更新し、不要な再起動を避ける。

## 常駐サービスも管理

crontab では定期タスクしか扱えないが、sdtab は `@service` で常駐サービスも管理できる。

```bash
sdtab add "@service" "uv run python app.py" \
  --name slack-bot \
  --env-file .env \
  --exec-start-pre "/home/ec2-user/.local/bin/uv sync" \
  --description "Slack Bot"
```

`Restart=always` がデフォルトなので、プロセスが落ちても自動復帰する。`--memory-max` や `--cpu-quota` でリソース制限もかけられる。

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
cargo install --git https://github.com/kok1eee/systemdtab
sdtab init
```

## AI エージェント対応

`sdtab init` は Claude Code のスキルファイルを自動インストールする。以降、どのプロジェクトからでも自然言語で操作できる：

```
You> /sdtab 毎朝9時にreport.pyを実行して
```

Claude Code が意図を解釈し、`sdtab add "@daily/9" "uv run ./report.py" --dry-run` で確認を求めた後、タイマーを作成する。

## 実際の移行

EC2 上で crontab 30 件 + 常駐サービス 5 件を sdtab に移行した。

```
NAME                        TYPE     SCHEDULE               COMMAND                                   STATUS
ambient-task-agent          service  @service               ambient-task-agent serve --port 3100      ● active
batch-download              service  @service               uv run python bolt_batch_download.py      ● active
  Batch Download Service - Kintone to Excel via Slack Bot
hikken-hikken               timer    0 9,13,17,21 * * *     start.sh hikken                           ○ Tue 2026-03-03 09:00:00 JST
  必見スケジュール - 通常必見
maruhan                     timer    @26th/11:30            start.sh                                  ○ Thu 2026-03-26 11:30:00 JST
  マルハン見積システム - 毎月26日
media-plusmade              timer    0 1 * * *              start.sh plusmade --cron-mode             ○ Wed 2026-03-04 01:00:00 JST
  メディア管理 - plusmade
salesdata-monthly-start     timer    @1st/8                 start.sh monthly-start                    ○ Wed 2026-04-01 08:00:00 JST
  売上データ更新 - 月初処理
...（計42件）
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

- cron 構文がそのまま使える（学習コストゼロ）
- 拡張構文で可読性アップ（`@mon/13`, `@1st/8`）
- 失敗通知、リソース制限、ログ管理が標準装備
- 設定のエクスポート/インポートで環境移行が楽
- 中身は素の systemd ユニットファイルなのでロックインなし

crontab に不満があるなら、試してみてほしい。

https://github.com/kok1eee/systemdtab
