# Replacing Inline SQL with Kysely: A Type-Safe Query Builder Refactor in Hono + MySQL

## Target Readers

Backend engineers who maintain a Node.js application backed by MySQL and want to move away from raw SQL strings without reaching for a heavy ORM. Prior familiarity with TypeScript and basic SQL is assumed.

## What This Article Covers

- Why the infrastructure layer of a TDD Todo App needed a query builder
- What Kysely is and how it differs from both raw SQL and full ORMs
- The concrete files that changed and the key design decisions made
- Two Kysely idioms worth knowing: `.$if()` for optional filters and the `sql` template tag inside `onDuplicateKeyUpdate`
- Verification results confirming zero regressions

This article does **not** cover database migrations, Kysely's schema migration utilities, or the Hono routing layer.

---

## Problem Background

The backend of this TDD Todo App is built with **Hono** (a lightweight Node.js framework) and **MySQL**, following a clean architecture pattern where repository implementations live in the `infrastructure/` layer. Before this refactor, all database access used `mysql2/promise` with raw SQL strings passed directly to `pool.execute()`:

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

This approach has several friction points:

- **No compile-time safety**: column names like `name`, `updatedAt`, `deletedAt` are bare strings. A typo silently passes TypeScript's type checker.
- **Parameter mismatch risk**: the positional `?` placeholders and the values array must stay in sync manually. An off-by-one error is invisible until runtime.
- **Conditional queries are messy**: building a `WHERE` clause that optionally excludes a specific `id` requires string concatenation or array manipulation.
- **Return types require manual casting**: `pool.execute<AppRow[]>()` works, but the shape of `AppRow` is declared separately from the actual SQL and nothing enforces that they match.

---

## Why Kysely

[Kysely](https://kysely.dev/) is a type-safe, fluent SQL query builder for TypeScript. Unlike ORMs (Prisma, TypeORM), Kysely does not abstract away SQL — it produces SQL that looks exactly like what you would write by hand. The key benefit is that the TypeScript type system enforces:

- table names must match a `Database` interface you define
- column names in `select`, `where`, and `values` must match the table's column types
- return types are inferred automatically from your query

Kysely is also dialect-pluggable. This project uses the `MysqlDialect` adapter backed by a `mysql2` connection pool.

---

## What Changed

Five files were touched in a single refactor commit (`2ca9757`):

| File | Change |
|---|---|
| `backend/src/infrastructure/db.ts` | **New** — `Database` interface + `createKysely()` factory |
| `backend/src/infrastructure/mysql-app-repository.ts` | **Rewritten** — raw SQL → Kysely fluent API |
| `backend/src/infrastructure/mysql-todo-repository.ts` | **Rewritten** — raw SQL → Kysely fluent API |
| `backend/src/infrastructure/mysql-registry.ts` | **Updated** — wires `createKysely()` instead of `createMysqlPool()` |
| `backend/src/infrastructure/mysql-client.ts` | **Deleted** — superseded by `db.ts` |

---

## Implementation Details

### 1. `db.ts`: Schema Interfaces and the Kysely Factory

The entry point for all database access is the new `db.ts` file. It does two things:

**Defines TypeScript interfaces that mirror the MySQL schema:**

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

The `Database` interface is the type parameter passed to `Kysely<Database>`. Once wired, every query in the codebase is checked against these definitions at compile time.

**Creates the Kysely instance with `MysqlDialect`:**

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

> **Important**: `MysqlDialect` expects the **callback-style** `createPool` from `mysql2` (the root package), **not** `createPool` from `mysql2/promise`. Using the promise variant here causes a runtime error. The old `mysql-client.ts` used `mysql2/promise`, so this is a subtle but intentional switch in the import path.

---

### 2. Repository Rewrites: Fluent API Replaces SQL Strings

Every `pool.execute(sql_string, params)` call was replaced with a Kysely method chain. Here is the `save()` method for the App repository as a representative example:

**Before:**
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

**After:**
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

The column names in `.values({...})` are now object keys — TypeScript verifies them against `AppTable`. The positional `?` placeholders are gone entirely.

---

### Design Decision A: `sql` Template Tag Inside `onDuplicateKeyUpdate`

The MySQL `ON DUPLICATE KEY UPDATE name = VALUES(name)` syntax tells MySQL to use the value that was about to be inserted. Kysely's `.onDuplicateKeyUpdate()` method expects typed column values, but `VALUES(name)` is a MySQL-specific SQL expression — not a plain value.

The solution is Kysely's `sql` tagged template literal, which lets you inject raw SQL fragments into an otherwise type-safe query:

```typescript
import { sql } from 'kysely';

.onDuplicateKeyUpdate({
  name: sql`VALUES(name)`,
  updatedAt: sql`VALUES(updatedAt)`,
  deletedAt: sql`VALUES(deletedAt)`,
})
```

This is a deliberate and documented escape hatch. It is narrowly scoped — only the right-hand side of those three columns uses raw SQL. The column name keys (`name`, `updatedAt`, `deletedAt`) are still type-checked. The `sql` tag properly sanitizes and formats the fragment for the target dialect.

---

### Design Decision B: `.$if()` for Optional Filters

The `existsActiveByName()` method checks whether an App with a given name already exists, optionally excluding one specific `id` (used during updates to allow keeping the same name):

**Before (raw SQL approach — builds query string dynamically):**
```typescript
let query = `SELECT EXISTS (
  SELECT 1 FROM App WHERE name = ? AND deletedAt IS NULL
  ${excludeId ? 'AND id != ?' : ''}
) AS \`exists\``;
const params = excludeId ? [name, excludeId] : [name];
const [rows] = await pool.execute<ExistsRow[]>(query, params);
```

**After:**
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

`.$if(condition, callback)` is a Kysely method that applies a query transformation only when `condition` is `true`. This keeps the query fully typed and readable without any string interpolation or conditional array building. The `!` non-null assertion inside the callback is safe because the `.$if` condition already guards against `undefined`.

Two other notable changes in this method:
- The `EXISTS (SELECT 1 ...)` pattern was replaced with `COUNT(id)`, which is simpler and equally efficient for this use case.
- `.executeTakeFirst()` replaces destructuring `[rows]` and `rows[0]` into a single, intention-revealing call.

---

### 3. Boolean ↔ Integer Conversion Preserved

MySQL stores `BOOLEAN` columns as `TINYINT(1)` (values `0` or `1`). The `TodoTable` interface reflects this reality:

```typescript
export interface TodoTable {
  // ...
  completed: number; // MySQL BOOLEAN → 0/1
}
```

The repository explicitly handles the conversion in both directions:

- **Write**: `completed: todo.completed ? 1 : 0`
- **Read**: `completed: row.completed !== 0`

This was present in the original implementation and was carried over unchanged. The comment in the interface makes the contract explicit for future maintainers.

---

### 4. Registry: One Line Change, Clean Wiring

`mysql-registry.ts` is where all infrastructure dependencies are composed. The change here is minimal:

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

Both repositories share the same `Kysely<Database>` instance, which in turn shares a single `mysql2` connection pool with a limit of 10 connections. The connection configuration path (`getMysqlConnectionConfig()`) was not changed.

---

## Points to Watch Out For

### `mysql2` vs `mysql2/promise` Import

`MysqlDialect` from `kysely` requires the callback-based `createPool` from `mysql2` (not `mysql2/promise`). Passing a promise-pool instance will compile without errors but fail at runtime when Kysely attempts to call internal pool methods it expects in callback form. Always import from `'mysql2'` when wiring the dialect.

### `fn.count` Returns a String at Runtime

Despite the generic `fn.count<number>('id')`, MySQL drivers often return count values as strings. The explicit `Number(result?.count ?? 0)` coercion in `existsActiveByName` handles this correctly. Skipping the `Number()` wrap and directly comparing with `> 0` would fail silently if the driver returns `"1"` instead of `1`.

### `.$if` Non-Null Assertions

Inside a `.$if` callback, TypeScript still considers the captured variable potentially `undefined` even though the condition guards against it. A `!` non-null assertion is required (`excludeId!`). This is a known ergonomic limitation of the pattern; the assertion is safe as long as the condition precisely mirrors the check.

---

## Verification

After the refactor, all automated checks passed without modification:

| Check | Result |
|---|---|
| `npm run typecheck` | ✅ 0 errors |
| `npm run lint` | ✅ 0 errors |
| `npm run test:unit` | ✅ 143 tests passed across 9 test files |

Because the repository layer is the only thing that changed — and because the TDD workflow had unit tests covering all repository-facing behavior through in-memory fakes — the test suite confirmed that the external behavior of every interactor and controller remained identical.

---

## Summary

This refactor replaced all raw `pool.execute(sql_string, params)` calls in the MySQL infrastructure layer with **Kysely's type-safe fluent API**. The concrete improvements are:

- **Compile-time safety**: table and column names are checked by TypeScript; typos become build errors, not runtime failures.
- **No positional parameter drift**: values are passed as named object keys, not ordered `?` placeholders.
- **Readable conditional queries**: `.$if()` eliminates string interpolation for optional `WHERE` clauses.
- **Targeted raw SQL escape hatch**: the `sql` template tag covers MySQL-specific syntax (`VALUES()` in `ON DUPLICATE KEY UPDATE`) without sacrificing type safety elsewhere in the query.
- **Smaller surface area**: `mysql-client.ts` was deleted; its responsibility was absorbed into `db.ts`, which also owns the schema type definitions.

The migration required no changes to application logic, controllers, or tests — a clean demonstration of the value of keeping infrastructure behind a repository interface.
