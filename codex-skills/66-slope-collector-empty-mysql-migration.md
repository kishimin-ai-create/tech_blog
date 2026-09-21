---
title: "空のMySQLへAlembicを適用し初期スキーマを検証する"
tags: [Python, MySQL, Alembic, データベース]
---

# 空のMySQLへAlembicを適用し初期スキーマを検証する

## 対象読者

既存データのないMySQLへ、アプリケーションのMigrationだけで初期スキーマを構築できるか確認したい開発者を対象にする。

## 結論

本番DBや開発用DBを変更せず、一時的なDocker MySQLへ`alembic upgrade head`を実行する。Migrationが空の状態から最後のリビジョンまで適用され、必要なテーブルと`alembic_version`が作成されたことを確認した。

## 空のDBを検証対象にする理由

既存DBへの更新が成功しても、初回セットアップに必要なMigrationが途中で抜けている可能性がある。空のDBへ適用すれば、リビジョンの依存関係と初期DDLを一度に確認できる。

今回の検証では、専用の一時スキーマを使い、アプリケーションの開発用データを読み書きしない境界を保った。

## 確認したスキーマ

Migration後のテーブル一覧には、次の5つが含まれていた。

| テーブル | 役割 |
| --- | --- |
| `alembic_version` | 適用済みリビジョンの記録 |
| `sources` | 取得元の設定を保持 |
| `entities` | 取得対象の安定識別子を保持 |
| `records` | 収集した本文レコードを保持 |
| `assets` | レコードに関連するアセットを保持 |

確認後は一時スキーマを削除した。したがって、この検証で開発用DBや本番DBのデータを変更していない。

## 検証結果と制約

`alembic upgrade head`による空MySQL検証に成功し、5つのアプリケーション関連テーブルと`alembic_version`を確認した。記録は`branch-plans/test-8-verify-mvp.md`へ残している。

この手順は初期スキーマの構築を確認するもので、既存データを含む本番移行、バックアップ復元、長時間運用を証明しない。接続情報はテスト専用の環境設定から渡し、リポジトリへ保存しない。

## まとめ

Migrationの検証では、既存DBを使って「更新できた」と判断するだけでは不十分である。一時的な空MySQLを作り、最後のリビジョンまで適用してテーブル一覧を確認し、検証後に削除することで、初期構築の失敗と開発データへの副作用を分離できる。

## 根拠

- `branch-plans/test-8-verify-mvp.md`
- `8ac459d docs: record MySQL migration verification`
- `app/migrations/versions/`

確認日: 2026年9月21日
