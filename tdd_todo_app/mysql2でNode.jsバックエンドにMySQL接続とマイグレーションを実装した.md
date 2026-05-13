# mysql2 で Node.js バックエンドに MySQL 接続とマイグレーションを実装した

## 対象読者

- Node.js + TypeScript + Hono でバックエンドを構築しているエンジニア
- mysql2 を初めて使う、またはインメモリ実装から MySQL に切り替えようとしているエンジニア

## この記事のスコープ

**扱うこと**

- mysql2 のプール設定と TypeScript 型付き
- `mysql2` の癖（`execute()` の制限、`BOOLEAN`/`DateTime` の戻り値型）
- `NODE_ENV=test` によるインメモリ/MySQL 自動切り替え設計
- `migrate.ts` の冪等マイグレーションランナー実装
- `fileURLToPath(import.meta.url)` によるパス解決
- `AppError` への `ErrorOptions.cause` 追加によるエラートレーサビリティ

**扱わないこと**

- MySQL サーバー自体のセットアップ手順
- ソフトデリートと UNIQUE INDEX の競合バグ（別記事参照）
- ESLint 修正（別記事参照）

---

## 背景

TDD Todo App（Node.js + TypeScript + Hono + Bun）のバックエンドはここまでインメモリリポジトリで動作していた。単体テストは通るが、永続化がない状態だった。今回 mysql2 を導入してインフラ層を MySQL 対応に切り替えた。

---

## パッケージ追加と .env 整備

```bash
npm install mysql2
```

`.env` は Laravel テンプレートから Node.js 用の最小構成に置き換えた。

```dotenv
NODE_ENV=development
PORT=3000

DB_HOST=your_host
DB_PORT=your_port
DB_DATABASE=your_db
DB_USERNAME=your_username
DB_PASSWORD=your_password
```

`.env` は `.gitignore` に記載されており、リポジトリには含まれない。

---

## MySQL プール実装（`mysql-client.ts`）

接続プールは `mysql2/promise` を使って作成する。

```typescript
import mysql from 'mysql2/promise';

export type MysqlPool = mysql.Pool;

/**
 * Creates a MySQL connection pool from environment variables.
 */
export function createMysqlPool(): MysqlPool {
  return mysql.createPool({
    host: process.env.DB_HOST ?? '127.0.0.1',
    port: Number(process.env.DB_PORT ?? '3306'),
    database: process.env.DB_DATABASE ?? 'TDDTodoAppDB',
    user: process.env.DB_USERNAME ?? 'root',
    password: process.env.DB_PASSWORD ?? '',
    timezone: '+00:00',
    waitForConnections: true,
    connectionLimit: 10,
  });
}
```

`timezone: '+00:00'` を設定しておくと、MySQL が DATETIME を UTC として扱うため、JS 側の Date との変換が一致する。

---

## mysql2 の癖 3 つ

mysql2 を使っていると TypeScript 上の型や挙動に独特のクセがある。実装中に踏んだ3点をまとめる。

### 1. `execute()` は `USE dbname` に使えない → `query()` を使う

`execute()` はプリペアドステートメントとして動作するため、`USE` や `CREATE DATABASE` のような DDL には使えない（プレースホルダーを持てないため）。

```typescript
// NG: execute() では動かない
await connection.execute(`USE \`${dbName}\``);

// OK: query() を使う
await connection.query(`USE \`${dbName}\``);
```

マイグレーションランナー内の `CREATE DATABASE` と `USE` は `query()` で呼び出した。

### 2. `BOOLEAN` カラムは `number` で返る → `!== 0` でキャスト

MySQL の `BOOLEAN` は内部的に `TINYINT(1)` であり、mysql2 は `number`（`0` または `1`）として返す。TypeScript の型に `boolean` と書いても、実際の値は数値で来る。

```typescript
type TodoRow = RowDataPacket & {
  completed: number; // BOOLEAN だが number で返る
};

function rowToTodo(row: TodoRow): TodoEntity {
  return {
    completed: row.completed !== 0, // number → boolean に変換
    // ...
  };
}
```

`Boolean(row.completed)` でも変換できるが、`!== 0` の方が意図が明確で安全。

### 3. `DATETIME` カラムは JS `Date` オブジェクトで返る → `.toISOString()`

mysql2 は `DATETIME` を JS の `Date` として返す。アプリ層のエンティティが ISO 8601 文字列（`string`）を期待している場合は `.toISOString()` で変換する。

```typescript
type AppRow = RowDataPacket & {
  createdAt: Date;   // DATETIME → Date で返る
  deletedAt: Date | null;
};

function rowToApp(row: AppRow): AppEntity {
  return {
    createdAt: row.createdAt.toISOString(),
    deletedAt: row.deletedAt ? row.deletedAt.toISOString() : null,
    // ...
  };
}
```

---

## `NODE_ENV=test` による自動切り替え設計

`index.ts` でテスト時はインメモリ、それ以外は MySQL を使うように分岐した。

```typescript
import { createBackendRegistry } from './infrastructure/registry';
import { createMysqlBackendRegistry } from './infrastructure/mysql-registry';
import type { Hono } from 'hono';

const isTest = process.env.NODE_ENV === 'test';

let honoApp: Hono;

if (isTest) {
  const registry = createBackendRegistry();     // インメモリ
  honoApp = registry.app;
} else {
  const registry = createMysqlBackendRegistry(); // MySQL
  honoApp = registry.app;
}

export default honoApp;
```

Vitest は実行時に `NODE_ENV=test` を自動でセットする。このため、テストファイルが `index.ts` を import すると自動的にインメモリリポジトリが選ばれる。CI で MySQL を起動せずに単体テストを通せるのが狙い。

`honoApp` の型を `ReturnType<typeof createMysqlBackendRegistry>['app']` のような特定実装依存にせず `Hono` 共通型にしているのは、将来どちらかの実装の `app` 型が変わったときに型エラーを正しく検出するため。

---

## マイグレーションランナー（`migrate.ts`）

`_migrations` テーブルで適用済みファイルを管理し、未適用の SQL ファイルだけを順番に実行する冪等な設計。

```typescript
async function migrate(): Promise<void> {
  const dbName = process.env.DB_DATABASE ?? 'TDDTodoAppDB';

  // DB名バリデーション（SQLインジェクション対策）
  if (!/^\w+$/.test(dbName)) {
    throw new Error(`Invalid DB_DATABASE value: "${dbName}"`);
  }

  const connection = await createConnection({ /* ... */ multipleStatements: true });

  try {
    await connection.query(
      `CREATE DATABASE IF NOT EXISTS \`${dbName}\` CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci`,
    );
    await connection.query(`USE \`${dbName}\``);

    // _migrations テーブルを作成（初回のみ）
    await connection.execute(`CREATE TABLE IF NOT EXISTS _migrations (...)`);

    // migrations/ ディレクトリの .sql ファイルをソートして適用
    const migrationsDir = fileURLToPath(new URL('../../migrations', import.meta.url));
    const sqlFiles = (await readdir(migrationsDir)).filter(f => f.endsWith('.sql')).sort();

    for (const file of sqlFiles) {
      const [rows] = await connection.execute<MigrationRow[]>(
        'SELECT name FROM _migrations WHERE name = ?', [file],
      );
      if (rows.length > 0) { console.log(`  skip   ${file}`); continue; }

      const sql = await readFile(join(migrationsDir, file), 'utf-8');
      await connection.query(sql); // DDL は query() を使う
      await connection.execute('INSERT INTO _migrations (name) VALUES (?)', [file]);
      console.log(`  apply  ${file}`);
    }
  } finally {
    await connection.end();
  }
}
```

`npm run migrate` を実行するたびに、未適用のファイルだけが実行される。既存環境への影響がなく、複数回実行しても安全。

### `fileURLToPath(import.meta.url)` でパス解決を実行ディレクトリ非依存に

当初 `process.cwd()` でマイグレーションディレクトリを解決していたが、`backend/` 以外から実行したときにパスが合わなくなる問題があった。

```typescript
// 修正前: 実行ディレクトリに依存
const migrationsDir = join(process.cwd(), 'migrations');

// 修正後: スクリプトファイル自身の位置を基準にする
import { fileURLToPath } from 'node:url';
const migrationsDir = fileURLToPath(new URL('../../migrations', import.meta.url));
```

`import.meta.url` は ESM でスクリプト自身の URL を返す。`new URL('../../migrations', import.meta.url)` とすることで、`migrate.ts` の位置（`src/infrastructure/`）からの相対パスで `migrations/` を指定できる。CI や monorepo ルートから実行しても正しく解決される。

---

## `AppError` への `cause` 追加

当初の `catch` ブロックは元エラーを捨てていた。

```typescript
// 修正前: 元エラーが消える
} catch {
  throw new AppError('REPOSITORY_ERROR', 'Repository operation failed');
}
```

DB 接続エラーや制約違反が起きても `REPOSITORY_ERROR` しか手がかりがなく、原因追跡が困難。`AppError` コンストラクタに `ErrorOptions`（`cause`）を追加して対応した。

```typescript
// app-error.ts
export class AppError extends Error {
  public constructor(code: AppErrorCode, message: string, options?: ErrorOptions) {
    super(message, options); // options.cause を Error 標準プロパティとして伝播
    this.name = 'AppError';
    this.code = code;
  }
}

// mysql-app-repository.ts
} catch (err: unknown) {
  throw new AppError('REPOSITORY_ERROR', 'Repository operation failed', { cause: err });
}
```

`Error` の第2引数に `{ cause: err }` を渡すと、`error.cause` でラップ前の例外を参照できる。Node.js 16.9.0 / ES2022 以降で標準対応している機能。

---

## マイグレーションファイル構成

```
backend/migrations/
  001_create_app_table.sql      — App テーブル作成
  002_create_todo_table.sql     — Todo テーブル作成（外部キー含む）
  003_drop_app_name_unique_index.sql — name の UNIQUE INDEX 削除（ソフトデリート対応）
```

`App` テーブルにはソフトデリート用の `deletedAt DATETIME NULL` を持たせ、`idx_app_deletedAt` インデックスを付けている。`name` カラムへの全件 UNIQUE INDEX は付けていない（理由は別記事参照）。

---

## まとめ

| 課題                                      | 対処                                       |
| ----------------------------------------- | ------------------------------------------ |
| mysql2 で `USE` が `execute()` で動かない | `query()` に切り替え                       |
| `BOOLEAN` が `number` で返る              | `!== 0` で boolean にキャスト              |
| `DATETIME` が `Date` で返る               | `.toISOString()` で string に変換          |
| テスト時に MySQL 不要                     | `NODE_ENV=test` でインメモリに自動切り替え |
| マイグレーションが実行ディレクトリ依存    | `fileURLToPath(import.meta.url)` に変更    |
| `catch` でエラーが消える                  | `AppError` に `{ cause: err }` を追加      |
| DB名インジェクション                      | `/^\w+$/` でバリデーション                 |
