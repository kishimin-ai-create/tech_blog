# Enforcing User Ownership for Apps and Todos in a Hono + Kysely API

## Target Readers

- Backend engineers building multi-user REST APIs with Hono
- Engineers looking for a clean pattern to add Bearer token authentication and per-resource authorization to an existing codebase
- Developers familiar with TypeScript who want to see how ownership checks fit into a layered (interactor / repository / controller) architecture

---

## Problem Background

The Todo API started as a single-user application where any client could create, read, update, or delete any App or Todo without any identity check. That model breaks the moment a second user signs up: all their apps would be visible to everyone, and any user could modify or delete another user's data.

Two concrete problems needed to be solved:

1. **Authentication** — every request to protected endpoints must carry a valid token, and the server must resolve that token to a real user.
2. **Authorization** — even after authentication passes, the service layer must confirm that the authenticated user actually owns the App (or the App that contains the Todo) before allowing the operation.

---

## Solution Overview

The fix touches every layer of the backend stack — database, repository, service, controller, and HTTP router — plus a small change on the frontend to attach the token automatically.

| Layer | Change |
|---|---|
| Database | `userId` column added to the `App` table |
| Repository | `listActive` → `listActiveByUserId`; `existsActiveByName` scoped to user; `UserRepository.findByToken` added |
| Service | `findAccessibleApp` (ownership check) in `AppInteractor`; `ensureAppBelongsToUser` in `TodoInteractor` |
| Error model | `FORBIDDEN` error code added |
| HTTP router | `authMiddleware` added to every protected route; `userId` propagated from context |
| Frontend | `customFetch` reads Bearer token from `localStorage` and injects it as `Authorization` header |

---

## Implementation Details

### 1. Database Migration

```sql
-- backend/migrations/004_add_userId_to_app.sql
ALTER TABLE App
  ADD COLUMN userId CHAR(36) NOT NULL DEFAULT '' AFTER id,
  ADD INDEX idx_app_userId (userId);
```

The column is placed immediately after `id` with a non-null constraint. An index on `userId` makes the per-user app listing query efficient.

### 2. AppEntity Model

```ts
// backend/src/models/app.ts
export type AppEntity = {
  id: string;
  userId: string;      // <-- added
  name: string;
  createdAt: string;
  updatedAt: string;
  deletedAt: string | null;
};
```

### 3. Repository Interface Updates

```ts
// backend/src/repositories/app-repository.ts
export interface AppRepository {
  save(app: AppEntity): Promise<void>;
  listActiveByUserId(userId: string): Promise<AppEntity[]>; // renamed + scoped
  findActiveById(id: string): Promise<AppEntity | null>;
  existsActiveByName(name: string, userId: string, excludeId?: string): Promise<boolean>; // userId added
}
```

`listActive()` became `listActiveByUserId()` to make it impossible to accidentally fetch all apps regardless of owner. `existsActiveByName` also takes `userId` so that two different users can create apps with the same name without triggering a conflict.

```ts
// backend/src/repositories/user-repository.ts
export interface UserRepository {
  findByEmail(email: string): Promise<UserEntity | null>;
  findByToken(token: string): Promise<UserEntity | null>; // <-- added
  save(user: UserEntity): Promise<void>;
}
```

`findByToken` is the lookup that the authentication middleware uses to resolve a raw Bearer token to a `UserEntity`.

### 4. In-Memory Repository Implementation

The in-memory implementation (used in tests) mirrors the contract:

```ts
// listActiveByUserId filters by userId and non-deleted
function listActiveByUserId(userId: string): Promise<AppEntity[]> {
  return Promise.resolve().then(() =>
    withRepositoryError(() =>
      [...storage.apps.values()]
        .filter(app => app.userId === userId && app.deletedAt === null)
        .map(cloneApp),
    ),
  );
}

// existsActiveByName now scopes the duplicate check to the same user
function existsActiveByName(
  name: string,
  userId: string,
  excludeId?: string,
): Promise<boolean> {
  return Promise.resolve().then(() =>
    withRepositoryError(() =>
      [...storage.apps.values()].some(
        app =>
          app.userId === userId &&
          app.deletedAt === null &&
          app.name === name &&
          app.id !== excludeId,
      ),
    ),
  );
}

// findByToken scans the user store for a matching token
function findByToken(token: string): Promise<UserEntity | null> {
  return Promise.resolve().then(() =>
    withRepositoryError(() => {
      const user = [...store.values()].find(u => u.token === token);
      return user ? { ...user } : null;
    }),
  );
}
```

### 5. FORBIDDEN Error Code

```ts
// backend/src/models/app-error.ts
export type AppErrorCode =
  | 'CONFLICT'
  | 'NOT_FOUND'
  | 'UNAUTHORIZED'
  | 'FORBIDDEN'       // <-- added
  | 'REPOSITORY_ERROR';
```

`UNAUTHORIZED` means "you did not prove who you are" (no or invalid token). `FORBIDDEN` means "we know who you are, but you do not own this resource." Keeping them separate allows the HTTP layer to return the correct status code (`401` vs `403`).

### 6. Ownership Checks in the Service Layer

**AppInteractor** — a single private helper, `findAccessibleApp`, replaces the old `findExistingApp`. It adds one extra check after the `NOT_FOUND` guard:

```ts
async function findAccessibleApp(input: {
  appId: string;
  userId: string;
}): Promise<AppEntity> {
  const app = await appRepository.findActiveById(input.appId);
  if (!app) throw new AppError('NOT_FOUND', 'App not found');
  if (app.userId !== input.userId) {
    throw new AppError('FORBIDDEN', 'Access denied');
  }
  return app;
}
```

Every mutating operation — `get`, `update`, `delete` — now calls `findAccessibleApp` instead of `findExistingApp`. The `create` operation sets `userId` on the new entity and passes it through `existsActiveByName` to scope duplicate detection.

**TodoInteractor** — Todos do not own a `userId` directly; they belong to an App. The guard `ensureAppBelongsToUser` is therefore the right place to enforce ownership:

```ts
async function ensureAppBelongsToUser(
  appId: string,
  userId: string,
): Promise<AppEntity> {
  const app = await appRepository.findActiveById(appId);
  if (!app) throw new AppError('NOT_FOUND', 'App not found');
  if (app.userId !== userId) throw new AppError('FORBIDDEN', 'Access denied');
  return app;
}
```

All five Todo operations (`create`, `list`, `get`, `update`, `delete`) call this guard before proceeding.

### 7. Bearer Token Middleware in Hono

The authentication middleware is registered on every protected route. It reads the `Authorization` header, strips the `Bearer ` prefix, and resolves the token to a user via `userRepository.findByToken`. On success it stores the user ID in the Hono context variable `userId` so controllers can read it.

```ts
// backend/src/infrastructure/hono-app.ts (excerpt)

type AppEnv = {
  Variables: {
    userId: string;
  };
};

async function authMiddleware(
  c: Context<AppEnv>,
  next: Next,
): Promise<Response | void> {
  const authHeader = c.req.header('Authorization');
  if (!authHeader?.startsWith('Bearer ')) {
    return c.json(
      { success: false, data: null, error: { code: 'UNAUTHORIZED', message: 'Authentication required' } },
      401,
    );
  }

  const token = authHeader.slice(7);
  const user = await userRepository.findByToken(token);
  if (!user) {
    return c.json(
      { success: false, data: null, error: { code: 'UNAUTHORIZED', message: 'Invalid or expired token' } },
      401,
    );
  }

  c.set('userId', user.id);
  await next();
}
```

The Hono generic `Hono<AppEnv>` ties the context variable type to the app, so `c.get('userId')` is fully type-safe across all route handlers without casts.

CORS `allowHeaders` was also updated to include `Authorization`:

```ts
allowHeaders: ['Content-Type', 'Authorization'],
```

Without this, browsers reject preflight requests when the frontend sends the auth header.

### 8. Controller and Route Plumbing

Controllers now accept `userId` as a parameter and forward it to the use-case input:

```ts
// app-controller create example (conceptual)
async create(body: unknown, userId: string): Promise<PresenterResult> { ... }
```

Route handlers extract `userId` from the Hono context and pass it down:

```ts
app.post('/apps', authMiddleware, async c =>
  toJsonResponse(c, await appController.create(await readRequestBody(c), c.get('userId'))),
);
```

### 9. Frontend — Automatic Token Injection

The centralized `customFetch` wrapper reads the auth state that was previously stored in `localStorage` under the key `auth`, extracts `token`, and injects it into every outgoing request:

```ts
// frontend/src/api/client.ts (excerpt)
const authRaw = localStorage.getItem('auth');
let auth: AuthState | null = null;
if (authRaw) {
  try { auth = JSON.parse(authRaw) as AuthState | null; } catch { auth = null; }
}

const headers: Record<string, string> = {
  ...(options?.headers as Record<string, string> | undefined),
};
if (auth?.token) {
  headers.Authorization = `Bearer ${auth.token}`;
}

const response = await fetch(`${BASE_URL}${url}`, { ...options, headers });
```

Because all API calls go through `customFetch`, this single change is enough to make every existing call authenticated — no changes needed in individual query/mutation hooks.

---

## Testing

Two new test scenarios were added:

- **Unauthenticated request → 401** — sending a request without an `Authorization` header must return `{ code: "UNAUTHORIZED" }` and status `401`.
- **Cross-user resource access → 403** — sending a valid token for user A while targeting a resource owned by user B must return `{ code: "FORBIDDEN" }` and status `403`.

Existing tests were updated to supply `userId` wherever the new input types require it. The in-memory repositories, used throughout the test suite, behave identically to the MySQL implementations but without any database dependency.

---

## Points to Watch Out For

### App name uniqueness is now per-user

Before this change, no two apps could share the same name across the entire system. After the change, uniqueness is scoped to the owning user. This is almost certainly the right product behavior, but it is a semantic change that existing data or integration tests may not anticipate.

### `findByToken` does a full scan in the in-memory implementation

The in-memory `findByToken` iterates over all users to find a token match. This is acceptable in tests, but the MySQL implementation should use an indexed column if the token field is not already indexed.

### Default `userId` in the migration

The migration sets `DEFAULT ''` for existing rows. Any existing App records will have an empty string as their `userId`. Depending on how production data was seeded, a follow-up data migration may be needed to backfill real user IDs.

### CORS must include `Authorization`

Forgetting `Authorization` in `allowHeaders` causes every browser preflight to fail with a CORS error before the token is even checked. The fix was included in this commit, but it is easy to miss when adding auth to an existing API.

---

## Summary

This commit closes the authorization gap in the Todo API by threading `userId` through every layer — from the database schema, through the repository interfaces and service-layer ownership checks, up to the Hono route handlers. The key design decisions were:

- **Keep authentication in the HTTP layer** (`authMiddleware`) and pass `userId` as a plain value into the service layer, keeping business logic free of HTTP concerns.
- **Keep authorization in the service layer** (`findAccessibleApp`, `ensureAppBelongsToUser`), where the domain rules live, rather than duplicating checks in controllers.
- **Separate `UNAUTHORIZED` from `FORBIDDEN`** to give clients and operators meaningful diagnostic signals.
- **Scope duplicate-name detection to the user** as a natural consequence of multi-tenancy.
- **Centralize token injection in `customFetch`** on the frontend so individual hooks require no changes.
