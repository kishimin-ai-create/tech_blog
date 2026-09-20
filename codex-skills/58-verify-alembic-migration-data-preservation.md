# 復元したMySQL DBでAlembic migrationのデータ保持を検証する

## はじめに

`alembic upgrade head`が成功しても、必要なデータがすべて残ったことまでは証明できない。今回は既存DBのdumpを一時DBへ復元し、migration前後の行数、backfill値、制約、アプリケーション動作を分けて確認した。

この記事では、不可逆なschema変更を既存DBへ適用する前に、実データ相当で何を観測するかを整理する。

## 前提・環境

- OS: Windows
- Database: MySQL
- Migration tool: Alembic
- 検証対象: 旧revisionから新revisionへのupgrade
- 方針: 開発DBへ適用する前に、dumpから復元した一時DBで試す

対象のDB名、接続情報、収集元を識別できる情報は記載しない。

## やってみた結果

一時DBでmigrationが完了し、migration前後で次の行数が一致した。

| 種別 | migration前 | migration後 |
| --- | ---: | ---: |
| 収集元設定 | 2 | 2 |
| 対象Entity | 60 | 60 |
| 記事レコード | 7,865 | 7,865 |
| 関連アセット | 40,410 | 40,410 |

すべての既存EntityにはNULLでない安定識別子が設定され、期待した`UNIQUE(source_id, external_key)`も作成された。

## 実装・検証

### Step 1: 適用前の開始状態を固定する

開発DBのdumpを一時DBへ復元し、次を記録した。

- Alembic revision
- 比較対象テーブルの行数
- 変更対象columnと制約の状態

migration後の件数だけでは欠落を検出できないため、適用前の値を比較基準として残した。

### Step 2: 一時DBでupgradeする

復元先を接続先に指定し、通常のmigrationコマンドを実行した。

```text
alembic upgrade head
```

最初の試行では、外部キーを支えるindexの削除順序が原因でMySQL Error 1553が発生した。DDL順序を修正後、同じ開始状態から作り直した一時DBで再実行した。

### Step 3: schemaを確認する

終了コードだけでなく、次のschema契約を確認した。

1. Alembic revisionが新しい値になった
2. 新しい識別子columnがNULL不可になった
3. `UNIQUE(source_id, external_key)`が存在した
4. 置換対象の古い一意制約がなくなった

### Step 4: データを比較する

migration前後で4種類の行数を比較し、すべて一致することを確認した。また、既存Entityへbackfillされた識別子がNULLでないことも確認した。

行数一致はデータ内容の完全一致を意味しない。そのため、今回の変更対象である識別子と一意制約を追加で検査した。

### Step 5: アプリケーション境界を確認する

migration後のDBを使ってアプリケーションを起動し、health確認と代表的な読み取りが成功することを確認した。

これにより、DDLの完了だけでなく、アプリケーションが新しいschemaを利用できるところまで観測した。

## つまずいたところ

実データ相当の検証を追加したことで、SQLiteのschemaテストでは見つからなかったMySQL固有のDDL順序問題が表面化した。

テストDBと対象DBエンジンでは制約の扱いが異なる。高速な自動テストは維持しつつ、DB固有のmigrationは対象エンジンへ復元したデータでも確認する必要があった。

## 学んだこと

### 成功条件を4層に分ける

migration検証では、次の観測を分けると不足が見つけやすい。

| 層 | 確認内容 |
| --- | --- |
| revision | Alembicが意図したrevisionへ到達したか |
| schema | column、NULL制約、index、一意制約が正しいか |
| data | 件数、backfill値、重複が想定どおりか |
| application | 起動、health、代表的な読み取りが成功するか |

### 不可逆migrationでは事前復元が重要になる

今回のmigrationは、安定識別子を古い表示名依存の制約へ安全に戻せないため、downgradeを拒否する設計だった。その場合、適用前dumpと一時DB検証が復旧計画の重要な一部になる。

ただし、dumpが存在するだけでは復旧可能とは言えない。実際に復元でき、開始状態を再構成できることまで確認する必要がある。

## まとめ

- `alembic upgrade head`の成功とデータ保持は別々に確認する
- revision、schema、data、applicationの4層で観測する
- migration前後で2、60、7,865、40,410行が保持された
- backfill値と新しい一意制約も個別に確認した
- 不可逆migrationは、復元可能なdumpと一時DBで適用前に検証する

確認日: 2026年9月20日
