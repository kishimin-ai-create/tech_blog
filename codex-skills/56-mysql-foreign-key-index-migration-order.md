# MySQL Error 1553をAlembicの制約変更順序から直す

## 結論

MySQLで外部キー列を含む一意制約を置き換えるときは、外部キーを支える新しいindexを作ってから古いindexを削除する。

今回のAlembic migrationは、`source_id`を含む古い一意制約を先に削除したため、MySQL Error 1553で停止した。新しい`UNIQUE(source_id, external_key)`を先に作り、古い制約を後から削除する順序へ変更すると、既存DBをmigrationできた。

## 発生した問題

表示名を識別子としていた既存schemaへ、変更されにくい`external_key`を追加するmigrationを実行した。SQLiteを使うschemaテストは成功していたが、既存のMySQL DBでは制約の削除時に失敗した。

### 症状

```text
MySQL Error 1553: Cannot drop index ... needed in a foreign key constraint
```

### 発生条件

- Database: MySQL
- Migration tool: Alembic
- 変更前: `source_id`を含む既存の一意制約が外部キー用indexも兼ねる
- 変更後: `UNIQUE(source_id, external_key)`へ置き換える

### 影響

既存DBが新しいrevisionへ到達できなかった。migration途中で止まるため、新しいschemaを前提にする処理も開始できない状態だった。

## 調査

### SQLiteだけではMySQL固有の制約を再現できなかった

既存のMediumテストは、SQLite上で次を確認していた。

- `external_key`が追加され、NULL不可になる
- 既存行へ代替キーが設定される
- 新しい一意制約が存在する
- 古い一意制約がなくなる

このテストはmigration後のschemaには有効だったが、MySQLがDDLを実行する途中で要求する「外部キーを支えるindexを常に残す」という条件を観測していなかった。

### DDL操作の順序を記録した

Alembicの操作を記録する小さなテストダブルを使い、`upgrade()`が呼ぶ操作を順番に保存した。修正前は、古い一意制約の削除が新しい一意制約の作成より先だった。

```python
actions: list[str] = []

class RecordingBatch:
    def create_unique_constraint(self, name, columns):
        actions.append("create_unique_constraint")

    def drop_constraint(self, name, *, type_):
        actions.append(f"drop_{type_}")

assert actions.index("create_unique_constraint") < actions.index("drop_unique")
```

このassertionは修正前に失敗し、MySQLで必要な操作順序を回帰テストとして固定できた。

## 原因

外部キーには参照元列を支えるindexが必要である。変更前のDBでは、削除対象の一意制約がその役割も担っていた。

migrationが古い制約を先に削除すると、次の一意制約が作られるまでの間、`source_id`を支えるindexがなくなる。MySQLはこの一時的に不正な状態を許可しないため、Error 1553を返した。

最終的なschemaだけを比較しても、この問題は見つからない。原因はDDLの途中状態にあった。

## 解決方法

新しい一意制約の作成と、古い一意制約の削除を入れ替えた。

```python
with op.batch_alter_table("entities") as batch:
    batch.create_unique_constraint(
        "uq_entities_source_external_key",
        ("source_id", "external_key"),
    )
    batch.drop_constraint("source_id", type_="unique")
```

新しい複合一意制約も`source_id`から始まるため、古いindexを削除する時点で外部キーを支えるindexが存在する。

## 動作確認

次の2段階で確認した。

1. 操作記録テストで、新しい一意制約の作成が古い制約の削除より先であることを確認した。
2. 既存データを復元したMySQL DBで、旧revisionから新revisionまでAlembic migrationを実行した。

プロジェクト全体では、型チェック、Lint、54件のテスト、コンテナbuildが成功した。復元DBでもmigrationは完了し、変更前後の行数が一致した。

## 再発防止

schemaの最終形だけでなく、DB固有のDDL順序もテストする。特に次の変更では、途中状態を確認する価値がある。

- 外部キー列を含むindexや一意制約の削除
- 主キーや参照先の変更
- NULL許可からNULL不可への変更
- 大量データのbackfillを挟む制約追加

操作記録テストは実DBテストの代替ではない。高速な回帰検出には操作記録を使い、最終確認は対象DBエンジンで行う、という分担にした。

## まとめ

- 症状: Alembic migrationがMySQL Error 1553で停止した
- 原因: 外部キーを支える古いindexを、新しいindexより先に削除していた
- 解決: `UNIQUE(source_id, external_key)`を作成してから古い制約を削除した
- 教訓: migrationは最終schemaだけでなく、DDLの途中状態と操作順序も検証する

確認日: 2026年9月20日
