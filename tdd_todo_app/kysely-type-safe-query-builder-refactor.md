# インライン SQL を Kysely に置き換える: Hono + MySQL のタイプセーフなクエリ ビルダー リファクタリング

## 対象読者

MySQL によってサポートされる Node.js アプリケーションを保守しており、重い ORM を使用せずに生の SQL 文字列から移行したいと考えているバックエンド エンジニア。 TypeScript と基本的な SQL に事前に精通していることが前提となります。

## この記事の内容

- TDD Todo アプリのインフラストラクチャ層にクエリ ビルダーが必要な理由
- Kysely とは何か、また生の SQL や完全な ORM との違い
- 変更された具体的なファイルと主要な設計上の決定
- 知っておく価値のある Kysely の 2 つのイディオム: オプションのフィルターの `.$if()` と、`onDuplicateKeyUpdate` 内の `sql` テンプレート タグ
- ゼロリグレッションを確認した検証結果

この記事では、データベース移行、Kysely のスキーマ移行ユーティリティ、または Hono ルーティング レイヤーについては**説明しません**。

---

## 問題の背景

この TDD Todo アプリのバックエンドは、**Hono** (軽量の Node.js フレームワーク) と **MySQL** で構築されており、リポジトリ実装が `infrastructure/` レイヤーに存在するクリーンなアーキテクチャ パターンに従っています。このリファクタリングが行われる前は、すべてのデータベース アクセスで `mysql2/promise` が使用され、生の SQL 文字列が `pool.execute()` に直接渡されていました。

```typescript
// Before: raw SQL in mysql-app-repository.ts
await pool.execute(
  `INSERT INTO App (id, name, createdAt, updatedAt, deletedAt)
   VALUES (?, ?, ?, ?, ?)
   ON DUPLICATE KEY UPDATE
     name      = VALUES(name),
     updatedAt = VALUES(updatedAt),
     deletedAt = VALUES(deletedAt)`,
  [app.id, app.name, new Date(app.createdAt), new Date(app.updatedAt), app.deletedAt ? new Date(app.deletedAt) : null],
);
```

このアプローチにはいくつかの摩擦点があります。

- **コンパイル時の安全性はありません**: `name`、`updatedAt`、`deletedAt` などの列名は裸の文字列です。タイプミスは、TypeScript の型チェッカーに黙って渡されます。
- **パラメーターの不一致のリスク**: 位置 `?` プレースホルダーと値の配列は手動で同期しておく必要があります。 off-by-one エラーは実行時まで表示されません。
- **条件付きクエリは厄介です**: オプションで特定の `id` を除外する `WHERE` 句を構築するには、文字列の連結または配列の操作が必要です。
- **戻り値の型には手動キャストが必要です**: `pool.execute<AppRow[]>()` は機能しますが、`AppRow` の形状は実際の SQL とは別に宣言されており、それらの一致を強制するものはありません。

---

## なぜカイセリーなのか

[Kysely](https://kysely.dev/) は、TypeScript 用のタイプセーフで流暢な SQL クエリ ビルダーです。 ORM (Prisma、TypeORM) とは異なり、Kysely は SQL を抽象化しません。手動で作成したものとまったく同じように見える SQL を生成します。主な利点は、TypeScript 型システムが次のことを強制することです。

- テーブル名は、定義した `Database` インターフェイスと一致する必要があります
- `select`、`where`、および `values` の列名はテーブルの列タイプと一致する必要があります
- 戻り値の型はクエリから自動的に推測されます

Kysely は方言プラグインも可能です。このプロジェクトは、`mysql2` 接続プールによってサポートされる `MysqlDialect` アダプターを使用します。

---

## 何が変わったのか

1 つのリファクタリング コミット (`2ca9757`) で 5 つのファイルが操作されました。

|ファイル |変更 |
|---|---|
| `backend/src/infrastructure/db.ts` | **新規** — `Database` インターフェイス + `createKysely()` ファクトリ |
| `backend/src/infrastructure/mysql-app-repository.ts` | **書き直されました** — 生の SQL → Kysely の流暢な API |
| `backend/src/infrastructure/mysql-todo-repository.ts` | **書き直されました** — 生の SQL → Kysely の流暢な API |
| `backend/src/infrastructure/mysql-registry.ts` | **更新** — `createMysqlPool()` の代わりに `createKysely()` を配線します。
| `backend/src/infrastructure/mysql-client.ts` | **削除** — `db.ts` に置き換えられました |

---

## 実装の詳細

### 1. `db.ts`: スキーマ インターフェイスと Kysely ファクトリ

すべてのデータベース アクセスのエントリ ポイントは、新しい `db.ts` ファイルです。次の 2 つのことを行います。

**MySQL スキーマをミラーリングする TypeScript インターフェイスを定義します:**

```typescript
export interface AppTable {
  id: string;
  name: string;
  createdAt: Date;
  updatedAt: Date;
  deletedAt: Date | null;
}

export interface TodoTable {
  id: string;
  appId: string;
  title: string;
  completed: number; // MySQL BOOLEAN stores as 0/1
  createdAt: Date;
  updatedAt: Date;
  deletedAt: Date | null;
}

export interface Database {
  App: AppTable;
  Todo: TodoTable;
}
```

`Database` インターフェイスは、`Kysely<Database>` に渡される type パラメーターです。配線されると、コードベース内のすべてのクエリがコンパイル時にこれらの定義に対してチェックされます。

**`MysqlDialect` を使用して Kysely インスタンスを作成します:**

```typescript
import { Kysely, MysqlDialect } from 'kysely';
import { createPool } from 'mysql2'; // ← callback-style pool, NOT mysql2/promise

export function createKysely(): Kysely<Database> {
  const config = getMysqlConnectionConfig();
  return new Kysely<Database>({
    dialect: new MysqlDialect({
      pool: createPool({
        host: config.host,
        port: config.port,
        database: config.database,
        user: config.user,
        password: config.password,
        timezone: '+00:00',
        waitForConnections: true,
        connectionLimit: 10,
      }),
    }),
  });
}
```

> **重要**: `MysqlDialect` は、`mysql2` (ルート パッケージ) からの **コールバック スタイル** `createPool` を期待します。**そうではありません**。 `mysql2/promise` からの `createPool`。ここで Promise バリアントを使用すると、実行時エラーが発生します。古い `mysql-client.ts` は `mysql2/promise` を使用していたため、これはインポート パスにおける微妙ではありますが意図的な切り替えです。

---

### 2. リポジトリの書き換え: SQL 文字列を Fluent API に置き換える

すべての `pool.execute(sql_string, params)` 呼び出しは Kysely メソッド チェーンに置き換えられました。代表的な例として、アプリ リポジトリの `save()` メソッドを次に示します。

**前に：**
```typescript
await pool.execute(
  `INSERT INTO App (id, name, createdAt, updatedAt, deletedAt)
   VALUES (?, ?, ?, ?, ?)
   ON DUPLICATE KEY UPDATE
     name      = VALUES(name),
     updatedAt = VALUES(updatedAt),
     deletedAt = VALUES(deletedAt)`,
  [app.id, app.name, new Date(app.createdAt), new Date(app.updatedAt), app.deletedAt ? new Date(app.deletedAt) : null],
);
```

**後：**
```typescript
await db
  .insertInto('App')
  .values({
    id: app.id,
    name: app.name,
    createdAt: new Date(app.createdAt),
    updatedAt: new Date(app.updatedAt),
    deletedAt: app.deletedAt ? new Date(app.deletedAt) : null,
  })
  .onDuplicateKeyUpdate({
    name: sql`VALUES(name)`,
    updatedAt: sql`VALUES(updatedAt)`,
    deletedAt: sql`VALUES(deletedAt)`,
  })
  .execute();
```

`.values({...})` の列名はオブジェクト キーになりました。TypeScript はそれらを `AppTable` に対して検証します。位置 `?` プレースホルダーは完全になくなりました。

---

### 設計上の決定 A: `onDuplicateKeyUpdate` 内の `sql` テンプレート タグ

MySQL `ON DUPLICATE KEY UPDATE name = VALUES(name)` 構文は、挿入しようとしている値を使用するように MySQL に指示します。 Kysely の `.onDuplicateKeyUpdate()` メソッドは型付きの列値を想定していますが、`VALUES(name)` は MySQL 固有の SQL 式であり、プレーンな値ではありません。

解決策は、Kysely の `sql` タグ付きテンプレート リテラルです。これを使用すると、生の SQL フラグメントを型安全なクエリに挿入できます。

```typescript
import { sql } from 'kysely';

.onDuplicateKeyUpdate({
  name: sql`VALUES(name)`,
  updatedAt: sql`VALUES(updatedAt)`,
  deletedAt: sql`VALUES(deletedAt)`,
})
```

これは意図的に文書化された脱出ハッチです。これは範囲が狭く、これら 3 つの列の右側のみが生の SQL を使用します。列名キー (`name`、`updatedAt`、`deletedAt`) は引き続き型チェックされます。 `sql` タグは、ターゲット方言に合わせてフラグメントを適切にサニタイズし、フォーマットします。

---

### 設計上の決定 B: オプションのフィルターの `.$if()`

`existsActiveByName()` メソッドは、指定された名前のアプリがすでに存在するかどうかを確認し、オプションで 1 つの特定の `id` (同じ名前を維持できるように更新中に使用されます) を除外します。

**前 (生の SQL アプローチ - クエリ文字列を動的に構築):**
```typescript
let query = `SELECT EXISTS (
  SELECT 1 FROM App WHERE name = ? AND deletedAt IS NULL
  ${excludeId ? 'AND id != ?' : ''}
) AS \`exists\``;
const params = excludeId ? [name, excludeId] : [name];
const [rows] = await pool.execute<ExistsRow[]>(query, params);
```

**後：**
```typescript
const result = await db
  .selectFrom('App')
  .select(({ fn }) => fn.count<number>('id').as('count'))
  .where('name', '=', name)
  .where('deletedAt', 'is', null)
  .$if(excludeId !== undefined, qb => qb.where('id', '!=', excludeId!))
  .executeTakeFirst();
return Number(result?.count ?? 0) > 0;
```

`.$if(condition, callback)` は、`condition` が `true` の場合にのみクエリ変換を適用する Kysely メソッドです。これにより、文字列補間や条件付き配列の構築を行わなくても、クエリが完全に型指定され、読み取り可能に保たれます。 `.$if` 条件がすでに `undefined` から保護されているため、コールバック内の `!` 非 null アサーションは安全です。

このメソッドには他に 2 つの注目すべき変更点があります。
- `EXISTS (SELECT 1 ...)` パターンは、この使用例ではよりシンプルで同等に効率的な `COUNT(id)` に置き換えられました。
- `.executeTakeFirst()` は、構造化された `[rows]` と `rows[0]` を単一の意図を明らかにする呼び出しに置き換えます。

---

### 3. ブール値 ↔ 整数変換を保持

MySQL は `BOOLEAN` 列を `TINYINT(1)`（値は `0` または `1`）として保存します。`TodoTable` インターフェイスはこの実態を反映しています。

```typescript
export interface TodoTable {
  // ...
  completed: number; // MySQL BOOLEAN → 0/1
}
```

リポジトリは、両方向の変換を明示的に処理します。

- **書き込み**: `completed: todo.completed ? 1 : 0`
- **読み取り**: `completed: row.completed !== 0`

これは元の実装に存在し、変更されずに引き継がれました。インターフェイス内のコメントにより、将来のメンテナに対して契約が明示的に示されます。

---

### 4. レジストリ: 1 行の変更、すっきりした配線

`mysql-registry.ts` は、すべてのインフラストラクチャの依存関係が構成される場所です。ここでの変更は最小限です。

```typescript
// Before
import { createMysqlPool } from './mysql-client';
const pool = createMysqlPool();
const appRepository = createMysqlAppRepository(pool);
const todoRepository = createMysqlTodoRepository(pool);

// After
import { createKysely } from './db';
const db = createKysely();
const appRepository = createMysqlAppRepository(db);
const todoRepository = createMysqlTodoRepository(db);
```

両方のリポジトリは同じ `Kysely<Database>` インスタンスを共有し、接続数が 10 に制限された単一の `mysql2` 接続プールを共有します。接続構成パス (`getMysqlConnectionConfig()`) は変更されませんでした。

---

## 注意すべき点

### `mysql2` と `mysql2/promise` インポート

`kysely` からの `MysqlDialect` には、`mysql2` (`mysql2/promise` ではない) からのコールバックベースの `createPool` が必要です。 Promise-pool インスタンスを渡すとエラーなしでコンパイルされますが、Kysely がコールバック形式で予期する内部プール メソッドを呼び出そうとすると実行時に失敗します。ダイアレクトを配線するときは、必ず `'mysql2'` からインポートしてください。

### `fn.count` は実行時に文字列を返します

汎用の `fn.count<number>('id')` にもかかわらず、MySQL ドライバーはカウント値を文字列として返すことがよくあります。 `existsActiveByName` の明示的な `Number(result?.count ?? 0)` 強制は、これを正しく処理します。ドライバーが `1` ではなく `"1"` を返す場合、`Number()` ラップをスキップして `> 0` と直接比較すると、サイレントに失敗します。

### `.$if` 非 Null アサーション

`.$if` コールバック内では、条件によって保護されている場合でも、TypeScript はキャプチャされた変数を潜在的に `undefined` とみなします。 `!` 非 null アサーションが必要です (`excludeId!`)。これは、パターンの人間工学上の既知の制限です。条件がチェックを正確に反映している限り、アサーションは安全です。

---

## 検証

リファクタリング後、すべての自動チェックは変更なしで合格しました。

|チェック |結果 |
|---|---|
| `npm run typecheck` | ✅ エラーは 0 |
| `npm run lint` | ✅ エラーは 0 |
| `npm run test:unit` | ✅ 9 つのテスト ファイルで 143 のテストに合格 |

変更されたのはリポジトリ層だけであるため、また、TDD ワークフローには、メモリ内のフェイクを通じてリポジトリに面したすべての動作をカバーする単体テストが含まれていたため、テスト スイートでは、すべてのインタラクターとコントローラーの外部動作が同一のままであることが確認されました。

---

## まとめ

このリファクタリングにより、MySQL インフラストラクチャ層の生の `pool.execute(sql_string, params)` 呼び出しがすべて **Kysely のタイプセーフな Fluent API** に置き換えられました。具体的な改善点は次のとおりです。

- **コンパイル時の安全性**: テーブル名と列名は TypeScript によってチェックされます。タイプミスは実行時のエラーではなく、ビルド エラーになります。
- **位置パラメータのドリフトなし**: 値は、順序付けされた `?` プレースホルダーではなく、名前付きオブジェクト キーとして渡されます。
- **読み取り可能な条件付きクエリ**: `.$if()` は、オプションの `WHERE` 句の文字列補間を排除します。
- **ターゲットを絞った生の SQL エスケープ ハッチ**: `sql` テンプレート タグは、クエリ内の他の場所の型安全性を犠牲にすることなく、MySQL 固有の構文 (`ON DUPLICATE KEY UPDATE` の `VALUES()`) をカバーします。
- **表面積が小さい**: `mysql-client.ts` が削除されました。その責任は、スキーマ タイプ定義も所有する `db.ts` に吸収されました。

移行では、アプリケーション ロジック、コントローラー、テストを変更する必要はありませんでした。これは、インフラストラクチャをリポジトリ インターフェイスの背後に維持することの価値を明確に示しました。
