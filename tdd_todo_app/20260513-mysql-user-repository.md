# Fixing a Silent Production Bug: In-Memory User Repository Wired to MySQL Registry

**Date:** 2026-05-13
**Commit:** `0546a19`
**Scope:** `backend/` — infrastructure layer, migrations

---

## Target Readers

- Backend engineers working with Hono / Node.js and a repository pattern
- Engineers who have encountered mysterious 401s or data loss after server restarts
- Anyone curious about applying TDD to an infrastructure layer fix

---

## The Bug: Every Restart Wiped All Users

After the authentication system was built out, a subtle but critical wiring mistake lived quietly in production:

```ts
// backend/src/infrastructure/mysql-registry.ts  (before fix)
import { createInMemoryUserRepository } from './in-memory-repositories';

// ...
const userRepository = createInMemoryUserRepository(); // ← BUG
```

The MySQL registry — the composition root used in the production server — was injecting an **in-memory user repository** instead of a MySQL-backed one. The `App` and `Todo` repositories both pointed at the real database, but `UserRepository` stored its data in a plain JavaScript `Map` that lived only for the duration of the Node.js process.

### Observed Symptoms

| Event | Result |
|---|---|
| User signs up | ✅ Succeeds (written to in-memory Map) |
| Server restarts (deploy, crash, scaling) | Map is destroyed |
| User tries to log in | ❌ `401 Unauthorized` — user simply does not exist |
| User tries to use an authenticated endpoint | ❌ `401 Unauthorized` — token lookup fails |

The failure was non-obvious because everything *appeared* to work on a freshly started server in development. It only became visible in production where restarts are routine.

---

## Why It Happened

The project follows a **ports-and-adapters (hexagonal) architecture**. The `UserRepository` interface lives in the domain layer:

```ts
// backend/src/repositories/user-repository.ts
export interface UserRepository {
  findByEmail(email: string): Promise<UserEntity | null>;
  findByToken(token: string): Promise<UserEntity | null>;
  save(user: UserEntity): Promise<void>;
}
```

During early development, an in-memory implementation of this interface was created for fast iteration and testing. That's perfectly reasonable. The mistake was that when `mysql-registry.ts` (the production composition root) was assembled, the MySQL adapter for `UserRepository` hadn't been written yet — and the in-memory version was used as a placeholder. The placeholder was never replaced.

There was no runtime error, no type error, and no test failure to surface this. Both implementations satisfy `UserRepository`, so TypeScript stayed silent.

---

## The Fix: TDD-Driven Implementation

The fix followed a strict Red → Green cycle to ensure correctness without spinning up a real database.

### Step 0: Migration — Create the `users` Table

Before writing any application code, a migration file was added to define the schema:

```sql
-- backend/migrations/005_create_user_table.sql
CREATE TABLE IF NOT EXISTS users (
  id VARCHAR(36) NOT NULL PRIMARY KEY,
  email VARCHAR(255) NOT NULL UNIQUE,
  token VARCHAR(36) NOT NULL UNIQUE,
  passwordHash VARCHAR(255) NOT NULL,
  createdAt DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

Key decisions:
- `id` and `token` are `VARCHAR(36)` — matching UUID v4 strings.
- `email` and `token` both carry `UNIQUE` constraints so the database enforces invariants independently of application logic.
- `IF NOT EXISTS` makes the migration safely re-runnable.

### Step 1: Extend the Kysely `Database` Type

Kysely's type-safety depends on the `Database` interface being complete. Without adding `users`, every query against the `users` table would be a type error:

```ts
// backend/src/infrastructure/db.ts (added)
export interface UserTable {
  id: string;
  email: string;
  token: string;
  passwordHash: string;
  createdAt: Date;
}

export interface Database {
  App: AppTable;
  Todo: TodoTable;
  users: UserTable;   // ← new
}
```

This is a pure type annotation change — zero runtime impact — but it unlocks compiler-checked queries throughout the repository implementation.

### Step 2: 🔴 RED — Write the Tests First

Eight unit tests were written against a not-yet-existing `createMysqlUserRepository` function. Running the test suite at this point produced import errors, confirming the correct RED state.

The test strategy used a **hand-rolled Kysely mock** — a chain of `vi.fn()` stubs that mimics the fluent Kysely builder API — so no real database connection was needed:

```ts
function makeMockDb() {
  const executeTakeFirst = vi.fn();
  const execute = vi.fn();
  const onDuplicateKeyUpdate = vi.fn(() => ({ execute }));
  const values = vi.fn(() => ({ onDuplicateKeyUpdate }));
  const insertInto = vi.fn(() => ({ values }));
  const where2 = vi.fn(() => ({ executeTakeFirst }));
  const where1 = vi.fn(() => ({ where: where2, executeTakeFirst }));
  const selectAll = vi.fn(() => ({ where: where1 }));
  const selectFrom = vi.fn(() => ({ selectAll }));

  return {
    db: { selectFrom, insertInto } as unknown as Parameters<
      typeof createMysqlUserRepository
    >[0],
    mocks: { /* all fns */ },
  };
}
```

The eight test cases covered:

| Method | Scenario |
|---|---|
| `findByEmail` | No row found → returns `null` |
| `findByEmail` | Row found → returns `UserEntity` |
| `findByEmail` | DB throws → rethrows as `AppError('REPOSITORY_ERROR')` |
| `findByToken` | No row found → returns `null` |
| `findByToken` | Row found → returns `UserEntity` |
| `findByToken` | DB throws → rethrows as `AppError('REPOSITORY_ERROR')` |
| `save` | Happy path → calls `insertInto('users')` with correct values |
| `save` | DB throws → rethrows as `AppError('REPOSITORY_ERROR')` |

### Step 3: 🟢 GREEN — Implement the Repository

With failing tests in place, `mysql-user-repository.ts` was created:

```ts
// backend/src/infrastructure/mysql-user-repository.ts (key excerpts)

function rowToUser(row: { id: string; email: string; token: string;
                          passwordHash: string; createdAt: Date }): UserEntity {
  return { id: row.id, email: row.email, token: row.token, passwordHash: row.passwordHash };
}

export function createMysqlUserRepository(db: Kysely<Database>): UserRepository {
  async function findByEmail(email: string): Promise<UserEntity | null> {
    try {
      const row = await db
        .selectFrom('users')
        .selectAll()
        .where('email', '=', email)
        .executeTakeFirst();
      return row ? rowToUser(row) : null;
    } catch (err) {
      throw new AppError('REPOSITORY_ERROR', 'Repository operation failed', { cause: err });
    }
  }

  async function save(user: UserEntity): Promise<void> {
    try {
      await db
        .insertInto('users')
        .values({ ...user, createdAt: new Date() })
        .onDuplicateKeyUpdate({
          email: sql`VALUES(email)`,
          token: sql`VALUES(token)`,
          passwordHash: sql`VALUES(passwordHash)`,
        })
        .execute();
    } catch (err) {
      throw new AppError('REPOSITORY_ERROR', 'Repository operation failed', { cause: err });
    }
  }

  // findByToken follows the same pattern as findByEmail
  return { findByEmail, findByToken, save };
}
```

A few implementation details worth noting:

**`rowToUser` strips `createdAt`** — `UserEntity` intentionally does not expose `createdAt`. The mapping function makes this boundary explicit rather than relying on object spread, which would silently include the extra column.

**`save` uses `ON DUPLICATE KEY UPDATE`** — This is a MySQL-specific upsert. Because `id` is the primary key, re-saving an existing user (e.g., updating the token after login) works without a separate SELECT-then-INSERT-or-UPDATE decision at the application level.

**All errors become `AppError('REPOSITORY_ERROR')`** — The domain layer only knows about `AppError` codes. Raw database errors are logged and then wrapped, keeping infrastructure concerns out of domain logic.

### Step 4: Wire the Real Repository

The one-line fix that resolved the production bug:

```ts
// backend/src/infrastructure/mysql-registry.ts
- import { createInMemoryUserRepository } from './in-memory-repositories';
+ import { createMysqlUserRepository } from './mysql-user-repository';

  const userRepository = createMysqlUserRepository(db);  // ← was createInMemoryUserRepository()
```

---

## Test Results

After the implementation:

```
✓ mysql-user-repository.small.test.ts (8 tests)
✓ Full suite: 89 unit tests passing
✓ TypeScript typecheck: clean
✓ Lint: clean
```

---

## Key Takeaways

### 1. The repository pattern does not prevent wiring mistakes
The `UserRepository` interface is satisfied by both the in-memory and MySQL implementations. TypeScript cannot tell you which one *should* be in production — that's a human judgment encoded in the composition root. Code reviews and integration tests are the safeguards.

### 2. Composition roots deserve scrutiny
`mysql-registry.ts` is the file that decides what actually runs in production. Any new domain component (like a repository) must be checked in every registry where it appears. A checklist or automated test that asserts "all repositories in `mysql-registry.ts` are MySQL-backed" would have caught this immediately.

### 3. TDD works well for infrastructure adapters
Because `UserRepository` is a well-defined interface, it was straightforward to mock the Kysely builder chain and write tests before the implementation existed. The mock chain looks verbose but gives precise control over each method call, which is exactly what you want when verifying that SQL queries are constructed correctly.

### 4. Error wrapping belongs in the adapter, not the use case
Wrapping raw DB errors into `AppError('REPOSITORY_ERROR')` at the repository layer keeps the use cases and domain logic free of MySQL-specific failure modes. If the database is swapped out in the future, the error contract stays the same.

---

## Summary

A placeholder in-memory repository was left wired into the production MySQL registry, causing all user data to vanish on every server restart and producing 401 errors for previously registered users. The fix involved writing a Kysely-backed `UserRepository` implementation using TDD — 8 unit tests written first, then the implementation created to make them pass — followed by a single import swap in the composition root. The full test suite of 89 unit tests passes with a clean typecheck and lint.
