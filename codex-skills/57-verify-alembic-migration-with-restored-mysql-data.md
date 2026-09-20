# GTID付きMySQL dumpを復元してAlembic migrationのデータ保持を検証する

## はじめに

Alembic migrationのテストが成功しても、開発DBに蓄積したデータを保持できるとは限らない。今回は、既存DBをdumpし、一時DBへ復元してからmigrationを実行し、テーブルごとの行数と新しい制約を確認した。

その過程で、GTID情報を含むdumpを同じMySQLサーバーへ復元すると失敗した。dump作成時に`--set-gtid-purged=OFF`を指定することで、一時DBを使った非破壊の検証を完了できた。

## 前提・環境

- OS: Windows
- Database: MySQL
- Migration tool: Alembic
- 検証対象: 旧revisionから新revisionへのupgrade
- 方針: 開発DBを直接使って最初の試行をせず、復元先の一時DBで確認する

DB名、接続情報、収集元を識別できる情報はこの記事では扱わない。

## やってみた結果

一時DBへ既存データを復元し、Alembic migrationを完了できた。migration前後で次の行数が保持された。

| 種別 | 行数 |
| --- | ---: |
| 収集元設定 | 2 |
| 対象Entity | 60 |
| 記事レコード | 7,865 |
| 関連アセット | 40,410 |

すべての既存EntityにはNULLでない安定識別子が設定され、期待した複合一意制約も作成された。

## 実装・検証

### Step 1: migration前のDBをdumpする

最初は通常のdumpを作成した。しかし、同じGTID有効サーバー上の一時DBへ復元すると、dumpに含まれるGTID_PURGEDの設定が既存サーバー状態と衝突して復元できなかった。

そこで、検証用dumpではGTID_PURGEDを出力しないようにした。

```text
mysqldump ... --set-gtid-purged=OFF --result-file=<backup.sql>
```

`<backup.sql>`、接続先、認証情報、DB名は環境に合わせる。コマンド履歴や記事へ秘密値を残さない。

### Step 2: 一時DBへ復元する

既存の開発DBとは別の一時DBを作り、修正したdumpを復元した。復元後、Alembicのrevisionがmigration前の値であることと、比較対象テーブルの行数を記録した。

この段階で行数を取得する理由は、migration後の件数だけを見ても欠落を検出できないためである。

### Step 3: 一時DBでupgradeする

復元先を接続先に指定して、通常のmigrationコマンドを実行した。

```text
alembic upgrade head
```

今回の最初のupgradeでは、外部キーを支えるindexの削除順序が原因でMySQL Error 1553が発生した。新しい複合一意制約を作成してから古い制約を削除するようmigrationを修正し、再度dumpから復元したDBで実行した。

### Step 4: schemaとデータを比較する

upgrade成功だけで完了とせず、次を確認した。

1. Alembicのrevisionが新しい値になった
2. migration前後で各テーブルの行数が一致した
3. 新しい識別子がNULLでない
4. `UNIQUE(source_id, external_key)`が存在する
5. アプリケーションのhealth確認と読み取り確認が成功する

既存DBへ適用する前に、復元済みデータでこれらを確認した。

## つまずいたところ

### 問題

GTID情報を含むdumpを、dump元と同じGTID有効MySQLサーバーへ復元できなかった。

### 原因

通常のdumpにはGTID_PURGEDを設定する文が含まれる場合がある。同じサーバーでは既存のGTID実行履歴があるため、復元時の設定と衝突した。

### 解決

一時DBへの復元を目的とするdumpでは、`--set-gtid-purged=OFF`を指定した。これにより、データとschemaを復元しつつ、サーバー全体のGTID履歴を書き換えない検証用dumpを作成できた。

この指定を本番のバックアップ方針へ無条件に適用するべきではない。レプリケーションや障害復旧ではGTID情報が必要になる場合があるため、用途ごとにdump方針を分ける必要がある。

## 学んだこと

### migrationの成功とデータ保持は別々に確認する

`alembic upgrade head`の終了コード0は、DDLが完了した証拠である。一方、必要な行がすべて残り、backfillされた値と制約が正しいことは別の検証になる。

今回のような識別子変更では、少なくとも次を分けて観測すると問題を切り分けやすい。

- schema: column、NULL制約、index、一意制約、revision
- data: migration前後の件数、backfill値、重複の有無
- application: health確認、代表的な読み取り

### 実データ相当の検証は一時DBで先に行う

開発DBを直接migrationすると、失敗時の切り戻しや原因調査が難しくなる。dumpから作った一時DBなら、失敗するたびに同じ開始状態を作り直せる。

今回のmigrationは不可逆であり、downgradeを安全に実装できない。そのため、適用前のdumpと一時DB検証がロールバック手段の一部になった。

## まとめ

- GTID付きdumpの同一サーバー復元は、GTID_PURGEDの設定で失敗する場合がある
- 検証用dumpでは`--set-gtid-purged=OFF`が有効だった
- migration前後の行数、backfill値、制約、アプリケーション動作を別々に確認した
- 2、60、7,865、40,410行の各データ群がmigration後も保持された
- 不可逆migrationでは、dumpと一時DBによる事前検証が特に重要になる

確認日: 2026年9月20日
