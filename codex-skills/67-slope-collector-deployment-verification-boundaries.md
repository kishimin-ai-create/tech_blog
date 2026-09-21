---
title: "外部取得を行わずにAPI・systemd・設定境界を検証する"
tags: [Python, Docker, GitHub Actions, systemd, セキュリティ]
---

# 外部取得を行わずにAPI・systemd・設定境界を検証する

## 対象読者

外部サイトへの実アクセスや本番DB変更を避けながら、収集アプリケーションのデプロイ前検証を組み立てたい開発者を対象にする。

## 結論

検証を「ローカルDockerで起動確認」「Linux CIでsystemdユニットの構文確認」「リポジトリの決定的スキャン」に分ける。今回の検証ではAPIのヘルスエンドポイント、CollectorのCLIヘルプ、systemdユニット、設定ファイル境界を確認したが、タイマー実行とjournald確認はデプロイ先へ残した。

## ローカルDockerで外部取得を始めない

Docker APIへ`GET /health`を送り、`200 {"status":"ok"}`を確認した。Collectorは`--help`を実行し、`collect-backfill`と`collect-daily`のコマンドが公開されていることだけを確認した。

CLIのヘルプ表示は、実際の収集を開始しない。したがって、ローカルの疎通確認と外部サイトへのHTTP負荷を分離できる。

## Linux CIでsystemdを構文検証する

Windows開発PCでは、実際のsystemdタイマー実行を完結できない。CIではUbuntu上にデプロイを想定したディレクトリ、専用ユーザー、仮想環境のPythonパスを準備し、次のユニットを`systemd-analyze verify`へ渡す。

```yaml
- name: Verify systemd units
  run: >-
    sudo systemd-analyze verify
    deploy/systemd/slope-collector.service
    deploy/systemd/slope-collector-backfill.service
    deploy/systemd/slope-collector.timer
```

これはユニットの参照先や構文を確認するCIゲートであり、タイマーを実際に待機させるテストではない。

## リポジトリへ秘密情報を持ち込まない

決定的なスキャンでは、実`.env`、秘密情報、取得先固有のURL・selector・識別子・fixtureが追跡対象へ入っていないことを確認した。追跡された環境ファイルは例示値に限定し、テストも匿名化した値を使う。

## 検証結果と未確認事項

Docker APIのヘルス確認、Collectorコマンド確認、Linux CIの`systemd-analyze verify`、リポジトリスキャンは完了した。Ruff、mypy、73テスト、カバレッジ87.12%、`git diff --check`も成功している。

一方、実Linux環境でのtimer起動とjournaldログ確認は未実施である。これらはステージングまたはデプロイ可能なLinux環境で、外部取得の対象と負荷を明示したうえで別途確認する必要がある。

## まとめ

デプロイ前検証を1つの「本番相当テスト」にまとめると、外部負荷を発生させる確認とローカルで安全にできる確認が混ざる。API、CLI、systemd構文、設定スキャンを独立したゲートに分け、実行環境が必要なtimerとjournaldだけをデプロイ先の課題として残すと、未確認範囲を隠さずに検証を進められる。

## 根拠

- `.github/workflows/ci.yml`
- `deploy/systemd/slope-collector.service`
- `deploy/systemd/slope-collector-backfill.service`
- `deploy/systemd/slope-collector.timer`
- `branch-plans/test-8-verify-mvp.md`
- `ba48b14 docs: record deployment boundary checks`
- `4c88b9f docs: record Linux systemd verification`

確認日: 2026年9月21日
