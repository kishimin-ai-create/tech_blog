# Deploying a Hono + React App to Render: Six Issues We Hit and How We Fixed Them

**Target readers**: Engineers who are deploying a Node.js + React full-stack app to Render for the first time and want a practical account of the friction points to expect.

**Stack**: React + Vite (Static Site), Hono + Node.js (Web Service), MySQL on Filess.io, Render Blueprint (`render.yaml`)

---

## Background

The TDD Todo App had been running locally with a MySQL database. The goal for this session was to promote it to a live environment on [Render](https://render.com/) using its Blueprint feature — a `render.yaml` file committed to the repository that provisions and auto-deploys all services from a single source of truth.

What looked like a straightforward config push turned into a six-stop debugging tour. Each issue was small in isolation, but together they cover almost every category of pain you can encounter in a first Render deployment: branch management, plan selection, build-time dependencies, cross-origin policy, database initialisation, and ORM date-type mismatch.

---

## Issue 1 — `render.yaml` existed only on a feature branch

### Symptom

Render Blueprint reads `render.yaml` from the default branch (`main`). The file had been created on the `frontend` branch and never merged, so Render could not find it.

### Fix

Merge the `frontend` branch to `main` via a pull request. Once the file landed on `main`, Render could read it and provision both services automatically.

### Takeaway

`render.yaml` must live on the branch Render watches. Check your repository's default branch before wiring up a Blueprint.

---

## Issue 2 — Render demanded a paid plan

### Symptom

After Blueprint detection, Render blocked provisioning and asked for a billing upgrade for the backend Web Service.

### Root cause

When `plan` is omitted in `render.yaml`, Render defaults to a paid instance type.

### Fix

Add `plan: free` explicitly to the backend service definition:

```yaml
# render.yaml
- type: web
  name: tdd-todo-app-backend
  env: node
  plan: free          # ← required to stay on the free tier
  buildCommand: cd backend && npm ci --include=dev && npm run build && node dist/infrastructure/migrate.mjs
  startCommand: cd backend && npm start
```

### Takeaway

Render's Blueprint spec does not assume the free tier. Always declare `plan: free` if that is your intent.

---

## Issue 3 — `esbuild` not found during the build

### Symptom

The build step failed with an error similar to:

```
sh: esbuild: not found
```

The backend uses `esbuild` to bundle `src/server.ts` and `src/infrastructure/migrate.ts` into `dist/`:

```json
"build:server": "esbuild src/server.ts --bundle --platform=node --packages=external --outfile=dist/server.mjs --format=esm"
```

### Root cause

Render sets `NODE_ENV=production` before running `buildCommand`. When `NODE_ENV=production`, `npm ci` skips `devDependencies` by default, and `esbuild` lives in `devDependencies`. Result: the bundler is absent at build time.

### Fix

Pass `--include=dev` to `npm ci` so that development dependencies are installed during the build phase:

```yaml
buildCommand: cd backend && npm ci --include=dev && npm run build && node dist/infrastructure/migrate.mjs
```

### Takeaway

`npm ci` respects `NODE_ENV`. Build tools (bundlers, type checkers, test runners) that live in `devDependencies` must be explicitly included when `NODE_ENV=production` is set by the platform. Alternatively, move only the build tools you need to `dependencies`, but `--include=dev` is the simpler global solution for a build step.

---

## Issue 4 — CORS errors blocked the frontend

### Symptom

After both services were running, the browser console showed:

```
Access to fetch at 'https://tdd-todo-app-backend.onrender.com/api/v1/apps'
from origin 'https://tdd-todo-app-frontend.onrender.com' has been blocked
by CORS policy: No 'Access-Control-Allow-Origin' header is present.
```

The frontend and backend are on different subdomains, so every API call was treated as a cross-origin request. The Hono app had no CORS middleware configured.

### Fix

Add `hono/cors` as a global middleware and drive the allowed origin through an environment variable:

```typescript
// backend/src/infrastructure/hono-app.ts
import { cors } from 'hono/cors';

export function createHonoApp(dependencies: HonoAppDependencies): Hono {
  const app = new Hono();

  app.use(
    '*',
    cors({
      origin: process.env.CORS_ORIGIN ?? '*',
      allowMethods: ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS'],
      allowHeaders: ['Content-Type'],
    }),
  );
  // ... routes
}
```

In the Render Dashboard, set the `CORS_ORIGIN` environment variable on the backend service to `https://tdd-todo-app-frontend.onrender.com`. Keeping the allowed origin in an environment variable avoids hardcoding a deployment-specific URL into source code, and makes it easy to restrict access properly in production while keeping `*` as a local fallback.

### Takeaway

Any time your frontend and backend are on different origins (even different subdomains of the same platform), the backend must explicitly emit CORS headers. Using an environment variable for the allowed origin is a clean way to handle different values per deployment environment.

---

## Issue 5 — App crashed on startup because the DB tables didn't exist

### Symptom

The backend started successfully, but the first API request returned a 500 error. The Render logs showed a MySQL error indicating that the `App` and `Todo` tables were missing.

### Root cause

The migration script (`dist/infrastructure/migrate.mjs`) was never executed on the Render MySQL instance. Locally, migrations were run manually with `npm run migrate`. There was no equivalent step in the cloud deployment pipeline.

### Fix

Append the migration script to the `buildCommand` so it runs once after the bundle is compiled, before the service starts:

```yaml
buildCommand: >
  cd backend &&
  npm ci --include=dev &&
  npm run build &&
  node dist/infrastructure/migrate.mjs
```

The migration script is idempotent — it tracks applied files in a `_migrations` table and skips any file it has already run. Running it on every deploy is therefore safe.

```typescript
// From migrate.ts — idempotency guard
const [rows] = await connection.execute<MigrationRow[]>(
  'SELECT name FROM _migrations WHERE name = ?',
  [file],
);
if (rows.length > 0) {
  console.log(`  skip   ${file}`);
  continue;
}
```

### Takeaway

If your migration tool is idempotent, running it as part of the deploy pipeline is the safest approach. It ensures the schema is always up to date without requiring a manual step or a separate Render Job.

---

## Issue 6 — MySQL rejected ISO 8601 datetime strings

### Symptom

Creating an App returned HTTP 500. The Render logs showed:

```
ER_TRUNCATED_WRONG_VALUE: Incorrect datetime value: '2026-05-06T12:01:53.756Z' for column 'createdAt' at row 1
```

### Root cause

The application's domain layer stores timestamps as ISO 8601 strings (e.g. `"2026-05-06T12:01:53.756Z"`). When these strings were passed directly as bind parameters to `mysql2`, MySQL's `DATETIME` column rejected the `T` separator and the `Z` suffix — it expects the format `YYYY-MM-DD HH:MM:SS`.

`mysql2` itself can serialise a JavaScript `Date` object into the correct format, but the repositories were passing raw ISO strings:

```typescript
// Before — passes ISO string directly
[app.id, app.name, app.createdAt, app.updatedAt, app.deletedAt]
```

### Fix

Wrap each ISO string in `new Date()` before passing it to the query. `mysql2` then serialises the `Date` object into a format MySQL accepts:

```typescript
// After — mysql-app-repository.ts
[
  app.id,
  app.name,
  new Date(app.createdAt),
  new Date(app.updatedAt),
  app.deletedAt ? new Date(app.deletedAt) : null,
]
```

The same fix was applied to `mysql-todo-repository.ts`:

```typescript
// After — mysql-todo-repository.ts
[
  todo.id,
  todo.appId,
  todo.title,
  todo.completed,
  new Date(todo.createdAt),
  new Date(todo.updatedAt),
  todo.deletedAt ? new Date(todo.deletedAt) : null,
]
```

Note that `null` must stay as `null` — wrapping it in `new Date()` would produce `Invalid Date`.

### Takeaway

MySQL's `DATETIME` type does not accept ISO 8601 format. If your domain model stores dates as strings, convert them with `new Date()` before passing them to `mysql2`. The driver handles the serialisation correctly from a `Date` object; it does not attempt to parse arbitrary string formats.

---

## Final `render.yaml`

After all six fixes, the Blueprint definition looks like this:

```yaml
services:
  # Frontend (Static Site)
  - type: web
    name: tdd-todo-app-frontend
    env: static
    buildCommand: cd frontend && npm ci && npm run build
    staticPublishPath: frontend/dist
    routes:
      - type: rewrite
        source: /*
        destination: /index.html

  # Backend (Web Service)
  - type: web
    name: tdd-todo-app-backend
    env: node
    plan: free
    buildCommand: cd backend && npm ci --include=dev && npm run build && node dist/infrastructure/migrate.mjs
    startCommand: cd backend && npm start
    envVars:
      - key: NODE_ENV
        value: production
      - key: DATABASE_URL
        sync: false
```

`DATABASE_URL` is marked `sync: false` so its value is never committed to the repository — it is set once in the Render Dashboard and stays there securely.

---

## Summary

| # | Problem | Root Cause | Fix |
|---|---------|------------|-----|
| 1 | Render couldn't find `render.yaml` | File was on a feature branch, not `main` | Merge to `main` |
| 2 | Render asked for a paid plan | `plan` key was absent; defaults to paid | Add `plan: free` |
| 3 | `esbuild` not found at build time | `npm ci` skips `devDependencies` when `NODE_ENV=production` | Use `npm ci --include=dev` |
| 4 | CORS blocked all API requests | No CORS headers on the backend | Add `hono/cors` middleware, configure via `CORS_ORIGIN` env var |
| 5 | Tables didn't exist on first deploy | Migration was never run in the cloud pipeline | Append `node dist/infrastructure/migrate.mjs` to `buildCommand` |
| 6 | MySQL rejected datetime values | ISO 8601 strings are not valid MySQL `DATETIME` input | Wrap strings with `new Date()` before passing to `mysql2` |

Each of these problems is common enough to appear in most first Render deployments. If you are deploying a similar stack, checking these six points before you push will likely save you several redeploy cycles.
