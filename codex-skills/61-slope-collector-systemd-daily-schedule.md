---
title: "日次収集と初回バックフィルをsystemd timerで分離する"
tags: [Python, systemd, スクレイピング, 運用]
---

# 日次収集と初回バックフィルをsystemd timerで分離する

## 対象読者

Python製の収集処理をsystemd timerで定期実行したい開発者を対象にする。

## 結論

日次処理と初回の全履歴取得は、同じ起動経路へ詰め込まない。日次処理は増分収集専用のコマンドとしてsystemd timerから起動し、初回バックフィルは明示的な手動操作に分離する。

今回の実装では、timerを毎日03:00（`Asia/Tokyo`）に設定し、実行漏れの取り戻しとランダム遅延を無効にした。これにより、定期処理の責務と時間のかかる初回処理の責務を分けられる。

## 背景

初回バックフィルは過去の履歴を広く走査するため、日次の増分処理より長くなる。これを定期実行へ設定すると、通常運用の1回の処理で意図せず大量のページを確認する可能性がある。

そこで、アプリケーションの入口を次の2つへ分けた。

| コマンド | 用途 | 起動方法 |
| --- | --- | --- |
| `collect-daily` | 前回チェックポイントからの増分収集 | systemd timer |
| `collect-backfill` | 初回の全履歴取得 | 手動実行 |

## timerの設定

```ini
[Timer]
OnCalendar=*-*-* 03:00:00 Asia/Tokyo
Persistent=false
AccuracySec=1min
RandomizedDelaySec=0
Unit=slope-collector.service
```

`Asia/Tokyo`をunitに明示することで、サーバーの既定タイムゾーンに依存しない。`Persistent=false`なので、停止中に時刻を過ぎても復旧直後に取り戻し実行しない。`AccuracySec=1min`は、指定時刻から1分程度の揺らぎを許容する設定である。

対象間の待ち時間や再試行はアプリケーション側の責務とし、timerへ追加のランダム遅延は設定していない。

## 手動バックフィルを分ける

初回の全履歴取得は、日次timerから呼び出さない。必要なときだけ、サービスユーザーと専用の仮想環境を指定して実行する。

```sh
cd /opt/slope-collector
sudo -u slope-collector /opt/slope-collector/.venv/bin/python -m app.collector collect-backfill
```

この分離により、定期実行の設定を変更せずに初回処理だけを監視・停止できる。どちらの処理もDBのチェックポイントを利用するが、開始条件と運用上の責務は同じではない。

## 検証

Issue #7の受け入れテストでは、次を確認した。

- `collect-daily`と`collect-backfill`がCLIヘルプに存在する。
- timerが毎日03:00（`Asia/Tokyo`）、`Persistent=false`、`AccuracySec=1min`、追加遅延なしである。
- timerが起動するunitが`slope-collector.service`である。
- 運用手順に有効化、状態確認、ログ確認、手動バックフィルの手順がある。

ローカル検証では、Ruff format、Ruff lint、mypy、pytest 67件、カバレッジ85.86%、`git diff --check`が成功した。systemd自体はWindows上で実行していないため、Linuxホストでのunit構文検証は未確認である。

## 制約

この変更はsystemd unitと運用手順をリポジトリへ追加したもので、実ホストへのデプロイ結果は含まない。初回バックフィルの完了をtimerが判定する仕組みも、この変更の範囲外である。

## まとめ

定期処理には増分収集だけを割り当て、初回全履歴取得は手動運用として分離する。timerには実行時刻と取り戻し方針を持たせ、対象単位間の待ち時間や再試行は収集処理へ委ねる。長時間処理を日次ジョブへ混ぜないことが、実行範囲を予測可能にする境界になる。

## 根拠

- `deploy/systemd/slope-collector.timer`
- `acceptance/test_issue_7_scheduling.py`
- `docs/operations/daily-collection.md`
- `f3d5873`〜`1f007e6`のIssue #7関連コミット

確認日: 2026年9月21日
