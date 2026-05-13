# MySQL 8.0 で `DROP INDEX IF EXISTS` が使えないマイグレーションを idempotent にする

## エラー概要

マイグレーション `003_drop_app_name_unique_index.sql` は当初、次の1行だけだった。

```sql
ALTER TABLE App DROP INDEX idx_app_name;
```

新規環境（初回セットアップや CI の fresh データベース）でこのマイグレーションを実行すると、  
`idx_app_name` インデックスが存在しないため次のエラーが発生し、マイグレーション全体が止まる。

```
Error: Can't DROP 'idx_app_name'; check that column/key exists
```

---

## 原因

### マイグレーション実行順序の問題

このリポジトリの `migrate.ts` はファイル名のアルファベット順にマイグレーションを実行する。

| ファイル | 内容 |
|---|---|
| `001_create_app_table.sql` | `App` テーブルを作成（インデックスなし） |
| `002_create_todo_table.sql` | `Todo` テーブルを作成 |
| `003_drop_app_name_unique_index.sql` | `idx_app_name` を削除 |

`idx_app_name` は `001` の初期スキーマには存在しない。  
これは別の作業（`002` と `003` の間のマイグレーション）で追加されたインデックスであり、  
新規環境には最初から存在しない。

したがって新規環境では `003` が実行されるタイミングで「存在しないインデックスを削除しようとしている」状態になる。

### MySQL 8.0 は `DROP INDEX IF EXISTS` をサポートしていない

PostgreSQL や MariaDB であれば次のように書くだけで解決できる。

```sql
-- PostgreSQL / MariaDB では動く
ALTER TABLE App DROP INDEX IF EXISTS idx_app_name;
```

しかし MySQL 8.0 は `ALTER TABLE ... DROP INDEX IF EXISTS` 構文を**サポートしていない**。  
実行すると構文エラーになる。

---

## 解決策：`information_schema` + Prepared Statement パターン

MySQL 8.0 でインデックスの存在を確認してから削除する標準的なパターンがある。  
`information_schema.STATISTICS` でインデックスの有無を検索し、  
結果に応じて実行する SQL を切り替えるという方法だ。

```sql
-- Drop idx_app_name only when it exists (idempotent for fresh databases).
-- MySQL 8.0 does not support DROP INDEX IF EXISTS, so we use a prepared
-- statement driven by information_schema to make this migration re-runnable.
SET @exist := (
  SELECT COUNT(*) FROM information_schema.STATISTICS
  WHERE table_schema = DATABASE()
    AND table_name   = 'App'
    AND index_name   = 'idx_app_name'
);
SET @sql := IF(@exist > 0, 'ALTER TABLE App DROP INDEX idx_app_name', 'SELECT 1');
PREPARE stmt FROM @sql;
EXECUTE stmt;
DEALLOCATE PREPARE stmt;
```

### 各ステップの説明

| ステップ | 役割 |
|---|---|
| `SET @exist := (SELECT COUNT(...))` | `idx_app_name` が存在するなら 1、しなければ 0 を変数に格納 |
| `SET @sql := IF(...)` | `@exist > 0` なら DROP 文、そうでなければ何もしない `SELECT 1` を変数に格納 |
| `PREPARE stmt FROM @sql` | 文字列変数から動的にプリペアドステートメントを作成 |
| `EXECUTE stmt` | プリペアドステートメントを実行 |
| `DEALLOCATE PREPARE stmt` | プリペアドステートメントを解放 |

インデックスが存在しない場合は `SELECT 1` が実行されるだけなので、エラーにならない。  
インデックスが存在する場合は通常通り `ALTER TABLE App DROP INDEX idx_app_name` が走る。

---

## 前提条件：`multipleStatements: true`

上記の SQL は複数のステートメントで構成されている。  
MySQL2 ドライバーはデフォルトで複数ステートメントの実行を禁止しているため、  
接続設定に `multipleStatements: true` が必要だ。

```typescript
// migrate.ts（概略）
const connection = await mysql.createConnection({
  host: process.env.DB_HOST,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  multipleStatements: true, // 複数ステートメントの実行を許可
});
```

このオプションが有効になっていれば、セミコロン区切りで複数の SQL を1回の `query()` で実行できる。

> **注意**: `multipleStatements: true` は SQL インジェクションのリスクを高める設定でもある。  
> マイグレーション専用の接続にのみ使い、アプリケーションのメイン接続では使わないようにすること。

---

## この修正で何が変わるか

| 状況 | 修正前 | 修正後 |
|---|---|---|
| 新規環境（`idx_app_name` なし） | エラーでマイグレーション停止 | `SELECT 1` が実行され正常完了 |
| 既存環境（`idx_app_name` あり） | インデックスを削除（正常動作） | インデックスを削除（変わらず） |
| CI の fresh データベース | マイグレーション失敗でテスト不能 | マイグレーション成功でテスト実行可能 |

---

## まとめ

MySQL 8.0 でマイグレーションをidempotent（べき等）にするには、  
`DROP INDEX IF EXISTS` が使えないため次のパターンを使う。

```
information_schema.STATISTICS で存在確認
  → IF() で実行する SQL を切り替え
    → Prepared Statement で動的実行
```

このパターンは `DROP INDEX` 以外にも、`ADD COLUMN` や `ADD CONSTRAINT` など  
「すでに存在する可能性があるオブジェクトを操作する」場面で応用できる。

マイグレーションをidempotentにしておくことは、CI/CD の安定性と  
新規開発者のセットアップ体験の両方に直結する。  
「初回実行でコケる」マイグレーションは、コードレビューで P1 相当の問題として扱うべき修正点だ。
