# ソフトデリートと UNIQUE INDEX の競合によるデータ破損バグを修正した

## エラー概要

ソフトデリート（`deletedAt` による論理削除）を実装している `App` テーブルに、`name` カラムへの全件 UNIQUE INDEX が存在していた。この組み合わせにより、削除済み App と同名の新規 App を作成しようとすると、**新規レコードの INSERT が失敗し、削除済みレコードが上書きされる**というデータ破損が発生する。

---

## 原因

### テーブル定義（修正前）

```sql
CREATE TABLE IF NOT EXISTS App (
  id        CHAR(36)     NOT NULL,
  name      VARCHAR(100) NOT NULL,
  -- ...
  deletedAt DATETIME     NULL DEFAULT NULL,
  PRIMARY KEY (id),
  UNIQUE INDEX idx_app_name (name),   -- ← 全行対象の UNIQUE INDEX
  INDEX idx_app_deletedAt (deletedAt)
);
```

`UNIQUE INDEX idx_app_name (name)` は `deletedAt` の値に関係なく、テーブル全行の `name` を一意にする。

### リポジトリ実装（修正前）

```typescript
async function save(app: AppEntity): Promise<void> {
  await pool.execute(
    `INSERT INTO App (id, name, createdAt, updatedAt, deletedAt)
     VALUES (?, ?, ?, ?, ?)
     ON DUPLICATE KEY UPDATE
       name      = VALUES(name),
       updatedAt = VALUES(updatedAt),
       deletedAt = VALUES(deletedAt)`,
    [app.id, app.name, app.createdAt, app.updatedAt, app.deletedAt],
  );
}
```

`ON DUPLICATE KEY UPDATE` は「重複キーが存在した場合に UPDATE する」構文。PRIMARY KEY（`id`）だけでなく、**UNIQUE INDEX（`name`）への重複でも発火する**。

### 何が起きるか

1. App A（id=`old-id`, name=`"my-app"`）を作成 → INSERT 成功
2. App A を削除 → `deletedAt` に日時をセット（行は残る）
3. App B（id=`new-id`, name=`"my-app"`）を新規作成しようとする
4. Service 層は `existsActiveByName("my-app")` を呼ぶ → `deletedAt IS NULL` の行のみ検索するため「衝突なし」と判定
5. `save(App B)` を呼ぶ
6. `INSERT ... ON DUPLICATE KEY UPDATE` が `name = "my-app"` の UNIQUE INDEX に引っかかる
7. **`old-id` の行の `name`/`updatedAt`/`deletedAt` が上書きされる**（`new-id` の行は作られない）
8. Service 層は `new-id` で App を作れたと認識するが、DB 上には `old-id` の行しかない
9. 以降の `findActiveById("new-id")` はすべて `null` を返す

---

## 修正内容

### 1. `UNIQUE INDEX` を `001_create_app_table.sql` から削除

```sql
-- 修正後
CREATE TABLE IF NOT EXISTS App (
  id        CHAR(36)     NOT NULL,
  name      VARCHAR(100) NOT NULL,
  createdAt DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updatedAt DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  deletedAt DATETIME     NULL     DEFAULT NULL,
  PRIMARY KEY (id),
  INDEX idx_app_deletedAt (deletedAt)  -- UNIQUE INDEX idx_app_name は削除
) ENGINE=InnoDB DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

### 2. 既存 DB 向けに `003_drop_app_name_unique_index.sql` を追加

マイグレーションランナーは冪等な連番ファイル管理のため、既存 DB のインデックスを除去するマイグレーションを別ファイルで追加した。

```sql
ALTER TABLE App DROP INDEX idx_app_name;
```

### 3. `save()` を INSERT/UPDATE 分岐に変更

`ON DUPLICATE KEY UPDATE` を廃止し、明示的な存在チェックで INSERT と UPDATE を切り替えた。

```typescript
async function save(app: AppEntity): Promise<void> {
  try {
    const [rows] = await pool.execute<ExistsRow[]>(
      'SELECT EXISTS(SELECT 1 FROM App WHERE id = ?) AS `exists`',
      [app.id],
    );
    if (rows[0].exists === 1) {
      await pool.execute(
        'UPDATE App SET name = ?, updatedAt = ?, deletedAt = ? WHERE id = ?',
        [app.name, app.updatedAt, app.deletedAt, app.id],
      );
    } else {
      await pool.execute(
        'INSERT INTO App (id, name, createdAt, updatedAt, deletedAt) VALUES (?, ?, ?, ?, ?)',
        [app.id, app.name, app.createdAt, app.updatedAt, app.deletedAt],
      );
    }
  } catch (err: unknown) {
    throw new AppError('REPOSITORY_ERROR', 'Repository operation failed', { cause: err });
  }
}
```

一意性の判定基準を `id`（PRIMARY KEY）のみにすることで、削除済みレコードの `name` との衝突が起きなくなる。名前の重複チェックは引き続きアプリ層の `existsActiveByName`（`deletedAt IS NULL` フィルタ付き）が担う。

---

## なぜ `Todo` テーブルは `ON DUPLICATE KEY UPDATE` のままか

`Todo` テーブルの `save()` は引き続き `ON DUPLICATE KEY UPDATE` を使っている。

```sql
INSERT INTO Todo (id, appId, title, completed, createdAt, updatedAt, deletedAt)
VALUES (?, ?, ?, ?, ?, ?, ?)
ON DUPLICATE KEY UPDATE
  title     = VALUES(title),
  completed = VALUES(completed),
  updatedAt = VALUES(updatedAt),
  deletedAt = VALUES(deletedAt)
```

`Todo` テーブルには `name` のような全件 UNIQUE INDEX がなく、PRIMARY KEY（`id`）のみが UNIQUE 制約。したがって `ON DUPLICATE KEY UPDATE` は純粋に「同一 `id` の upsert」として動作し、競合の問題は発生しない。

---

## まとめ

| 原因 | 全行対象の `UNIQUE INDEX (name)` がソフトデリートと共存できない |
|------|------|
| 症状 | `ON DUPLICATE KEY UPDATE` が削除済み行を上書きし、新規 ID が DB に作られない |
| 修正 | UNIQUE INDEX 削除 ＋ `save()` を明示的 INSERT/UPDATE 分岐に変更 |

ソフトデリートを採用する場合、全行対象の UNIQUE INDEX はほぼ必ず問題になる。MySQL では部分インデックス（`WHERE` 句付き）がサポートされないため、一意性チェックをアプリ層で担う設計が現実的な選択肢となる。
