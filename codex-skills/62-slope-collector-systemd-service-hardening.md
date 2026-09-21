---
title: "systemd oneshotサービスで収集処理の権限と観測性を分離する"
tags: [Python, systemd, セキュリティ, 運用]
---

# systemd oneshotサービスで収集処理の権限と観測性を分離する

## 対象読者

外部HTTPとDBへ接続するバッチを、専用ユーザーとjournaldで運用したい開発者を対象にする。

## 結論

収集処理をsystemdから起動するときは、専用の非rootユーザー、外部環境ファイル、`Type=oneshot`、基本hardening、journald出力を組み合わせる。systemdの責務を「1回の実行単位と権限・ログの境界」に限定し、収集リトライや失敗の詳細はアプリケーション側に残す。

## サービス定義

Issue #7では、サービスを次のように定義した。

```ini
[Unit]
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
User=slope-collector
Group=slope-collector
WorkingDirectory=/opt/slope-collector
EnvironmentFile=/etc/slope-collector/collector.env
ExecStart=/opt/slope-collector/.venv/bin/python -m app.collector collect-daily
Restart=no
TimeoutStartSec=infinity
UMask=0077
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
```

`Type=oneshot`により、timerから起動される1回の処理を1つのサービス実行として扱う。同じサービスが実行中の場合に別プロセスを起動する仕組みを追加せず、systemdのunit単位の実行制御を使う。

## 権限と設定ファイルを分ける

アプリケーションは`/opt/slope-collector`へ配置し、サービスユーザーは`slope-collector`とする。接続情報やデプロイ先固有の設定はGit管理対象のunitへ埋め込まず、`/etc/slope-collector/collector.env`から読み込む。

環境ファイルは`root:slope-collector`が所有し、モード`0640`で管理する。rootだけが書き換え、サービスユーザーは読み取りだけを行える境界である。実際の値は記事やリポジトリへ掲載しない。

`NoNewPrivileges=true`、`PrivateTmp=true`、`ProtectSystem=strict`は、収集処理が必要とする権限とファイルシステムの範囲を狭めるための設定である。これらの設定がアプリケーションの必要権限を満たすかは、デプロイ先で確認する必要がある。

## ログと失敗通知の責務

サービスの標準出力・標準エラーはjournaldへ送る。運用者は次のコマンドでtimerと実行結果を確認できる。

```sh
sudo systemctl status slope-collector.timer
sudo systemctl status slope-collector.service
sudo journalctl -u slope-collector.service
```

systemdには`OnFailure`の追加通知を設定していない。収集処理の一時的なリトライと失敗概要の通知は、Issue #6で実装したアプリケーション側の責務である。サービスは自動再起動せず、アプリケーション内のリトライと次回timer実行で再開する。

## 検証

受け入れテストで、次の設定契約を固定した。

- 専用ユーザー、作業ディレクトリ、環境ファイル、仮想環境の実行パス
- `Type=oneshot`、`Restart=no`、`TimeoutStartSec=infinity`
- `network-online.target`への依存
- `NoNewPrivileges`、`PrivateTmp`、`ProtectSystem`のhardening
- journald、手動起動、timer有効化を含む運用手順

ローカルでは、67件のテストが成功し、カバレッジは85.86%だった。Windows環境のため、Linux上での`systemd-analyze verify`、実際の権限・journald出力、外部接続を含む動作は未確認である。

## トレードオフ

基本hardeningは権限境界を明確にする一方、DBソケットや書き込みディレクトリなど、実行環境固有の許可が不足すると起動に失敗する可能性がある。`TimeoutStartSec=infinity`は長い収集をsystemdが途中終了させないが、処理時間の上限を設ける仕組みではない。長時間処理の分割や進捗管理は別の設計課題として残る。

## まとめ

systemd unitには、実行ユーザー、設定ファイルの場所、ファイルシステム制限、ログ経路を明示する。収集処理のリトライや失敗内容までsystemdへ重ねず、unitは安全な実行境界、アプリケーションは収集固有の制御という責務分担にすると、運用時の確認箇所を分けられる。

## 根拠

- `deploy/systemd/slope-collector.service`
- `acceptance/test_issue_7_scheduling.py`
- `docs/operations/daily-collection.md`
- `272276b`および`1f007e6`のIssue #7関連コミット

確認日: 2026年9月21日
