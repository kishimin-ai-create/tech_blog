# Two GitHub Actions Workflow Bugs Fixed: Missing MySQL Service Container and `node --env-file`

**Type:** Technical Issue Resolution  
**Commits:** `98a7490`, `3087fd6`  
**Target readers:** Engineers running Vitest integration tests against a live MySQL database in GitHub Actions, or anyone using Node.js to run a migration script on a remote server from a deploy workflow.

---

## Overview

Two bugs in separate GitHub Actions workflows were independently breaking CI coverage
runs and production database migrations. They came from the same class of failure:
**a workflow step that depended on database credentials or infrastructure assumed those
preconditions were already in place — they were not.**

This post documents each bug: what failed, why, and the minimal correct fix.

---

## Fix 1 — MySQL Service Container Missing from `test-coverage.yml`

### Symptom

Every run of the `Test Coverage` workflow ended at:

```
Fail when coverage failed
exit 1
```

The `Run backend integration coverage` step was consistently marked as failed, but
there were no visible test assertion errors in the output. The test runner could not
connect to anything before it even started.

### Root Cause

`backend/vitest.integration.config.ts` targets tests under `src/tests/integrations/`,
which open a live MySQL connection. The workflow ran `npm run coverage:integration`
without a `services:` block — meaning no MySQL container was ever started — and
without setting any `DB_*` environment variables. The mysql2 connection pool failed
immediately on every run.

The pattern that caused the permanent red:

```yaml
# Before — no services block on the job, no DB env vars on the step
- name: Run backend integration coverage
  id: backend_integration_coverage
  continue-on-error: true        # failure is caught, not propagated immediately
  run: npm run coverage:integration
  working-directory: backend
  # ← DB_HOST, DB_PORT, DB_DATABASE, DB_USERNAME, DB_PASSWORD all undefined

- name: Fail when coverage failed
  if: steps.backend_integration_coverage.outcome == 'failure'
  run: exit 1                    # fires every single time
```

`continue-on-error: true` is intentional here — it lets coverage artifacts upload even
when a threshold is missed. But with a missing MySQL connection, there was never a real
coverage result to upload. The `exit 1` gate fired unconditionally.

### The Fix (commit `98a7490`)

Three things were added to `.github/workflows/test-coverage.yml`.

**1. A `services.mysql` block on the job:**

```yaml
services:
  mysql:
    image: mysql:8.0
    env:
      MYSQL_ROOT_PASSWORD: ""
      MYSQL_ALLOW_EMPTY_PASSWORD: "yes"
      MYSQL_DATABASE: TDDTodoAppDB
    ports:
      - 3306:3306
    options: >-
      --health-cmd="mysqladmin ping -h 127.0.0.1 --silent"
      --health-interval=10s
      --health-timeout=5s
      --health-retries=5
```

GitHub Actions service containers share the same network namespace as the job runner.
The MySQL container is reachable at `127.0.0.1` on the mapped port — no hostname
resolution or bridge network required.

The `--health-cmd` block is critical. Without it, subsequent steps can start before
MySQL has finished initializing and is ready to accept connections, producing spurious
`connection refused` errors. GitHub Actions will wait until the health-check passes
before moving to the first step when `options` are configured this way.

**2. `DB_*` environment variables on both coverage steps:**

```yaml
- name: Run backend unit coverage
  id: backend_unit_coverage
  continue-on-error: true
  run: npm run coverage:unit
  working-directory: backend
  env:
    DB_HOST: 127.0.0.1
    DB_PORT: 3306
    DB_DATABASE: TDDTodoAppDB
    DB_USERNAME: root
    DB_PASSWORD: ""

- name: Run backend integration coverage
  id: backend_integration_coverage
  continue-on-error: true
  run: npm run coverage:integration
  working-directory: backend
  env:
    DB_HOST: 127.0.0.1
    DB_PORT: 3306
    DB_DATABASE: TDDTodoAppDB
    DB_USERNAME: root
    DB_PASSWORD: ""
```

The unit coverage step also receives DB credentials because some unit test paths
indirectly exercise mysql2 initialization code that reads connection config at import
time. If those env vars are absent, the process can fail before any test runs.

Env vars are scoped to each step rather than the job level. Keeping them at step scope
follows the principle of least privilege — even for credentials that are CI-only and
carry no real secret value here.

**3. A `Build backend` step and a `Run database migrations` step before integration tests:**

```yaml
- name: Build backend
  run: npm run build
  working-directory: backend

- name: Run database migrations
  run: node dist/infrastructure/migrate.mjs
  working-directory: backend
  env:
    DB_HOST: 127.0.0.1
    DB_PORT: 3306
    DB_DATABASE: TDDTodoAppDB
    DB_USERNAME: root
    DB_PASSWORD: ""
```

The migration script is compiled by esbuild from `src/infrastructure/migrate.ts` to
`dist/infrastructure/migrate.mjs`. Running `npm ci` alone does not produce this file.
The build step must precede the migration step, and the migration step must precede the
integration test step — the schema must exist before the tests run.

### Resulting Step Order

```
Checkout
→ Setup Node.js
→ Install dependencies (frontend / backend)
→ Run frontend coverage              (continue-on-error)
→ Run backend unit coverage          (continue-on-error + DB env)   ★ env added
→ Build backend                                                       ★ new
→ Run database migrations            (DB env)                         ★ new
→ Run backend integration coverage   (continue-on-error + DB env)   ★ env added
→ Generate combined coverage dashboard
→ Upload artifacts
→ Fail when coverage failed
```

---

## Fix 2 — Production Migration Not Loading `.env` in `deploy.yml`

### Symptom

After a push to `main`, the `Run database migrations` step SSHed into the VPS and
executed — but authentication failed or targeted the wrong database. The SSH session
itself succeeded; the issue was inside the Node.js process running on the server.

### Root Cause

The deploy workflow ran migrations like this:

```yaml
# Before
- name: Run database migrations
  env:
    SSH_USER: ${{ secrets.SAKURA_USER }}
    SSH_HOST: ${{ secrets.SAKURA_HOST }}
  run: |
    ssh "$SSH_USER@$SSH_HOST" \
      "cd ${{ env.DEPLOY_PATH }}/backend && node dist/infrastructure/migrate.mjs"
```

On the VPS, database credentials live exclusively in
`/var/www/tdd-todo-app/backend/.env`. This file is never committed to the repository
and is never rsynced by the workflow — it is a one-time server setup artifact.

When `node dist/infrastructure/migrate.mjs` runs, the migration script reads DB
connection config from `process.env`. Nothing loaded the `.env` file, so
`process.env.DB_HOST`, `process.env.DB_PASSWORD`, etc. were all `undefined`. The mysql2
driver fell back to compiled-in defaults, causing authentication failures against the
production MySQL instance.

This contrast makes the bug clear:

| Process | How `.env` is loaded |
|---|---|
| Application server (pm2) | `node_args: '--env-file=/var/www/tdd-todo-app/backend/.env'` in `ecosystem.config.cjs` |
| Migration script (before fix) | Nothing — env vars undefined, driver uses defaults |

The server process had been given `--env-file` from the start (covered in the prior
deploy pipeline article). The migration step had no equivalent.

### The Fix (commit `3087fd6`)

A single flag added to the `node` invocation:

```yaml
# After
- name: Run database migrations
  env:
    SSH_USER: ${{ secrets.SAKURA_USER }}
    SSH_HOST: ${{ secrets.SAKURA_HOST }}
  run: |
    ssh "$SSH_USER@$SSH_HOST" \
      "cd ${{ env.DEPLOY_PATH }}/backend && node --env-file .env dist/infrastructure/migrate.mjs"
```

`--env-file` is a built-in Node.js 20+ flag that reads key-value pairs from the
specified file into `process.env` before the script starts executing. No additional
packages (`dotenv`, `dotenv-cli`, etc.) are needed — Node.js handles it natively.

The path `.env` is relative to the working directory set by
`cd ${{ env.DEPLOY_PATH }}/backend`, so it resolves to
`/var/www/tdd-todo-app/backend/.env` — the same credentials file the pm2 process
already loads.

The migration script itself required no changes.

### Why Not Use Step-Level `env:` Like in `test-coverage.yml`?

In the coverage workflow, the Actions runner itself forks the Node.js process, so
GitHub Actions step-level `env:` injects variables directly into that process's
environment — it works perfectly.

In the deploy workflow, the Node.js process runs on the remote VPS as a child of the
SSH session. Step-level `env:` on the Actions runner does not cross the SSH boundary.
The credentials must already reside on the server. `--env-file` is the right mechanism
to load them into a process that the runner does not directly spawn.

---

## Summary

| Bug | Workflow | Root Cause | Fix |
|---|---|---|---|
| Integration tests always fail | `test-coverage.yml` | No MySQL container; no DB env vars; no compiled migration script; no schema | Add `services.mysql` with health-check; set `DB_*` env vars on coverage steps; add `Build backend` + `Run database migrations` steps |
| Production migration uses wrong credentials | `deploy.yml` | `node migrate.mjs` ran on VPS without loading the server-side `.env` | Add `--env-file .env` to the `node` invocation |

Both bugs share the same failure class: a step that depended on database connectivity
assumed that credentials and infrastructure were already in scope. They were not.

**Checklist when adding a database-dependent step to a workflow:**

1. **For runner-local execution:** Is a `services:` container defined? Does it have a `--health-cmd` to delay steps until the DB is ready?
2. **Connection config:** Are `DB_*` environment variables explicitly set at the step level?
3. **Build artifacts:** Is the migration script compiled before it is invoked?
4. **For remote execution (SSH):** Does the remote process have a mechanism to load credentials from a server-side file — e.g., `node --env-file .env`?

The fix for each case is narrow: provide what that specific execution context can
actually access, rather than assuming ambient configuration that was never wired up.
