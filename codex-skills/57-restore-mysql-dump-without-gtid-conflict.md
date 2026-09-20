# GTID付きMySQL dumpを同一サーバーへ復元できないときの対処

## 結論

GTID情報を含むMySQL dumpを、dump元と同じGTID有効サーバーへ復元すると、既存のGTID実行履歴と衝突する場合がある。

今回の一時DB検証では、dump作成時に`--set-gtid-purged=OFF`を指定し、GTID_PURGEDを出力しないことで復元できた。ただし、これは検証用dumpの判断であり、レプリケーションや障害復旧用のバックアップへ無条件に適用する設定ではない。

## 発生した問題

Alembic migrationを既存データ相当で試すため、開発DBをdumpし、同じMySQLサーバー上の一時DBへ復元しようとした。

### 症状

データを投入する前に、dumpに含まれるGTID_PURGEDの設定が既存サーバー状態と衝突し、復元が停止した。

### 発生条件

- Database: GTIDを有効にしたMySQL
- dump元と復元先: 同一MySQLサーバー
- 目的: 別名の一時DBでmigrationを検証する

DB名、接続情報、収集元を識別できる情報はこの記事では扱わない。

### 影響

一時DBをmigration前の状態へ復元できず、既存データを使ったupgrade検証を開始できなかった。

## 調査

通常のdumpには、取得元のGTID情報を復元先へ設定する文が含まれる場合がある。同じサーバーには既存のGTID実行履歴があるため、別DBへの復元であってもサーバー全体のGTID状態と衝突する。

今回の目的はレプリカや新規サーバーの構築ではなく、同一サーバー内の一時DBへschemaとデータを複製することだった。そのため、GTID履歴の復元は検証に不要だった。

## 原因

DBのデータ領域は分かれていても、GTID実行履歴はMySQLサーバー単位で管理される。dump内のGTID_PURGED設定は、復元先DBだけではなくサーバー全体の状態へ作用する。

同一サーバー上で一時DBを作る用途と、GTIDを引き継いで別サーバーを復元する用途を、同じdump設定で扱ったことが衝突の原因だった。

## 解決方法

検証用dumpの作成時に、GTID_PURGEDを出力しないオプションを追加した。

```text
mysqldump ... --set-gtid-purged=OFF --result-file=<backup.sql>
```

`<backup.sql>`、接続先、認証情報、DB名は環境に合わせる。秘密値をコマンド履歴や記事へ残さない。

このdumpを一時DBへ復元すると、サーバー全体のGTID履歴を設定せず、schemaとデータを検証用に複製できた。

## 動作確認

修正したdumpを同一MySQLサーバー上の一時DBへ復元し、次を確認した。

- 復元処理が完了した
- Alembic revisionがmigration前の値だった
- 比較対象テーブルの行数を取得できた
- 復元先で`alembic upgrade head`を実行できた

## 再発防止

dumpの用途ごとにGTID方針を明示する。

| 用途 | GTIDの扱い |
| --- | --- |
| 同一サーバー内の一時DB検証 | `--set-gtid-purged=OFF`を検討する |
| 別サーバーへの障害復旧 | 復旧設計に基づきGTIDを保持するか判断する |
| レプリケーション構築 | topologyとMySQL運用手順に従う |

`OFF`を常に安全な既定値とは扱わない。復元後に必要なGTID状態は用途によって異なる。

## まとめ

- 症状: GTID付きdumpを同じMySQLサーバーへ復元できなかった
- 原因: dumpのGTID_PURGED設定がサーバー既存のGTID履歴と衝突した
- 解決: 一時DB検証用dumpでは`--set-gtid-purged=OFF`を使った
- 教訓: dump設定はDB単位ではなく、サーバー全体のGTID用途から決める

確認日: 2026年9月20日
