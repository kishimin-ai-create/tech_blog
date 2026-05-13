# Automated Deployment to Sakura Cloud VPS with GitHub Actions

**Type:** Devlog / Release Note  
**Date:** 2026-05-04  
**Target readers:** Engineers who want to set up a push-to-deploy pipeline for a Node.js + Vite app on a Linux VPS, or anyone tracing how this project's production deployment works.

---

## Overview

This session wired up a fully automated deployment pipeline for the TDD Todo App — a React/Vite frontend backed by a Hono/Node.js API and MySQL database — hosted on a Sakura Cloud VPS.

The result: every push to `main` builds both ends of the stack, runs the test suite as a gate, rsyncs the artifacts to the server, backs up the database, runs pending migrations, and reloads the Node.js process via pm2, all without touching the server manually.

Five commits were involved:

| SHA | Description |
|---|---|
| `22f3dc8` | Initial deploy workflow + pm2 ecosystem config + esbuild scripts |
| `5c30648` | Security hardening: pinned host key, test gate, secret isolation, DB backup |
| `efad45e` | Move `eslint-plugin-unused-imports` to `devDependencies` |
| `98f089d` | Set `NODE_ENV=production` in pm2 config and document path coupling |
| `9bf6795` | Add dispositions and replies to the code review document |

---

## What Gets Deployed and How

### Frontend — Vite → rsync → Nginx

```
frontend/src  →  npm run build  →  frontend/dist/
                                        ↓
                              rsync --delete  →  /var/www/tdd-todo-app/frontend/
                                        ↓
                                   served by Nginx
```

`rsync -avz --delete` ensures stale files from a previous build are removed on every deploy.

### Backend — TypeScript → esbuild → rsync → pm2

Two esbuild commands compile the TypeScript source to ESM `.mjs` bundles:

```jsonc
// backend/package.json
"build:server":   "esbuild src/server.ts   --bundle --platform=node --packages=external --outfile=dist/server.mjs   --format=esm",
"build:migrate":  "esbuild src/infrastructure/migrate.ts --bundle --platform=node --packages=external --outfile=dist/infrastructure/migrate.mjs --format=esm"
```

`--packages=external` is the key flag: npm dependencies are **not** bundled into the output. Instead, `package.json` and `package-lock.json` are rsynced alongside the `dist/` directory, and `npm ci --omit=dev` installs only production dependencies on the server. This keeps the bundle small while preserving the normal `node_modules` resolution path.

### pm2 Process Management

`backend/ecosystem.config.cjs` is the pm2 config file rsynced to the server on every deploy:

```js
module.exports = {
  apps: [{
    name: 'tdd-todo-app',
    script: './dist/server.mjs',
    cwd: '/var/www/tdd-todo-app/backend',
    node_args: '--env-file=/var/www/tdd-todo-app/backend/.env',
    instances: 1,
    autorestart: true,
    watch: false,
    max_memory_restart: '300M',
    env: { NODE_ENV: 'production' },
  }],
};
```

The restart step uses idempotent logic — `pm2 reload` if the process already exists, `pm2 start` on first deploy:

```yaml
- name: Restart backend service
  run: |
    ssh "$SSH_USER@$SSH_HOST" << 'EOF'
      set -e
      if pm2 describe tdd-todo-app > /dev/null 2>&1; then
        pm2 reload tdd-todo-app --update-env
      else
        pm2 start /var/www/tdd-todo-app/backend/ecosystem.config.cjs
      fi
      pm2 save
    EOF
```

`pm2 reload` performs a rolling restart with zero downtime (connections are handed off before the old process exits). `--update-env` ensures any new env entries from the ecosystem config take effect.

### Database Migration

Before migrations run, a timestamped `mysqldump` snapshot is taken:

```yaml
- name: Backup database before migration
  run: |
    ssh "$SSH_USER@$SSH_HOST" \
      "mysqldump --defaults-file=${{ env.DEPLOY_PATH }}/backend/.my.cnf \
        TDDTodoAppDB > /var/backups/tdd-todo-app-$(date +%Y%m%d%H%M%S).sql"

- name: Run database migrations
  run: |
    ssh "$SSH_USER@$SSH_HOST" \
      "cd ${{ env.DEPLOY_PATH }}/backend && node dist/infrastructure/migrate.mjs"
```

The `.my.cnf` file (a one-time server setup step) holds the MySQL credentials so they never appear as plain CLI arguments in process listings.

---

## Security Hardening — Code Review Findings and Fixes

A code review of the initial commit (`22f3dc8`) identified six issues across two priority levels. All were resolved before merging.

### P1: ssh-keyscan is a TOFU vulnerability

**Before:**
```yaml
- name: Add server to known_hosts
  run: ssh-keyscan -H ${{ secrets.SAKURA_HOST }} >> ~/.ssh/known_hosts
```

`ssh-keyscan` trusts whatever key the server returns at scan time. An attacker who can hijack the DNS or BGP route at that moment gets their key trusted for every `ssh` and `rsync` call in the job.

**After:** The server's public key is stored once (obtained from a trusted network) in a GitHub Secret `SAKURA_KNOWN_HOST` and echoed directly:

```yaml
- name: Add server to known_hosts
  env:
    SAKURA_KNOWN_HOST: ${{ secrets.SAKURA_KNOWN_HOST }}
  run: |
    mkdir -p ~/.ssh
    echo "$SAKURA_KNOWN_HOST" >> ~/.ssh/known_hosts
```

This pins the expected fingerprint and eliminates the TOFU window on every deploy.

### P1: No test gate — broken code could reach production

The initial workflow went straight from `npm run build` to SSH deployment. A commit that broke tests would still be deployed.

**Fix:** Inline test steps were added between install and build for both sides:

```yaml
- name: Run frontend tests
  working-directory: frontend
  run: npm test

- name: Run backend unit tests
  working-directory: backend
  run: npm run test:unit
```

Integration tests are intentionally excluded here — they require a live MySQL instance that is not available on the Actions runner. Unit tests are the appropriate gate for a deploy workflow. Cross-workflow `needs:` (depending on the separate `backend.yaml` CI job) is not possible in GitHub Actions for jobs in different workflow files, so the tests are inline.

### P1: Migration ran with no prior database backup

Migrations contain irreversible DDL. Without a backup, a destructive `ALTER` or `DROP` would cause permanent data loss with no recovery path.

**Fix:** The `mysqldump` backup step described above was added immediately before the migration step.

### P2: Secrets interpolated directly into shell strings

Patterns like `ssh ${{ secrets.SAKURA_USER }}@${{ secrets.SAKURA_HOST }}` expand secret values directly into the shell command. If a secret value ever contained shell metacharacters (`;`, `` ` ``, `$(...)`), they would execute on the Actions runner.

**Fix:** All ssh/rsync steps now expose secrets via a step-level `env:` block and reference them as shell variables:

```yaml
- name: Deploy frontend
  env:
    SSH_USER: ${{ secrets.SAKURA_USER }}
    SSH_HOST: ${{ secrets.SAKURA_HOST }}
  run: |
    rsync -avz --delete frontend/dist/ "$SSH_USER@$SSH_HOST:${{ env.DEPLOY_PATH }}/frontend/"
```

### P2: `eslint-plugin-unused-imports` was in `dependencies`

This lint-only package had no runtime use but was listed under `dependencies`, so `npm ci --omit=dev` would install it (and its transitive tree) on every production deploy, wasting disk space and install time on a constrained VPS.

**Fix:** Moved to `devDependencies`. Production installs are now leaner.

### P2: `NODE_ENV` was never guaranteed to be `production`

If an operator copied `.env.example` to `.env` without adding `NODE_ENV=production`, the server process would run with `NODE_ENV` undefined. Libraries such as Hono gate production-safe error handling on `NODE_ENV === 'production'`.

**Fix:** Added `env: { NODE_ENV: 'production' }` to the pm2 app config. The pm2 `env` block takes precedence over `--env-file`, so production mode is guaranteed regardless of what the operator wrote in `.env`.

### P3: Path coupling between `ecosystem.config.cjs` and `deploy.yml`

`ecosystem.config.cjs` hard-codes `/var/www/tdd-todo-app/backend` in both `cwd` and `node_args`. The workflow defines the same root as `env.DEPLOY_PATH`. If one is changed without the other, pm2 will fail with a path-not-found error.

**Fix:** Added prominent `// IMPORTANT: must match DEPLOY_PATH in .github/workflows/deploy.yml` comments on both lines, and a `PATH COUPLING` block in the file-level JSDoc. Template generation was considered but deferred.

---

## Required GitHub Secrets

| Secret | Value |
|---|---|
| `SAKURA_SSH_PRIVATE_KEY` | SSH private key (Ed25519 or RSA) |
| `SAKURA_HOST` | Server hostname or IP |
| `SAKURA_USER` | SSH username |
| `SAKURA_KNOWN_HOST` | Pinned server public key — run `ssh-keyscan -H <server>` once from a trusted network |

All secrets are scoped to the `production` environment in GitHub (`environment: production` in the workflow), so they are only accessible during deploy runs.

---

## One-Time Server Setup

Before the first deploy, the VPS needs:

1. **Node.js 20+**, npm, and pm2 (`npm install -g pm2`)
2. **MySQL** running and accessible
3. `/var/www/tdd-todo-app/backend/.env` — copy from `.env.example` and fill in DB credentials
4. `/var/www/tdd-todo-app/backend/.my.cnf` — MySQL credentials for `mysqldump` (avoids plaintext passwords in process listings)
5. `pm2 startup && pm2 save` — registers pm2 as a systemd service so the process survives server reboots
6. **Nginx** configured to serve `frontend/` as static files and reverse-proxy `/api` to the backend port

---

## Full Workflow at a Glance

```
push to main
  │
  ├── Install frontend deps  →  Run frontend tests  →  Build frontend (Vite)
  ├── Install backend deps   →  Run backend unit tests  →  Build backend (esbuild)
  │
  ├── Setup SSH agent + add pinned known_hosts
  │
  ├── rsync frontend/dist/  →  /var/www/tdd-todo-app/frontend/
  ├── rsync backend/dist/ + migrations/ + package*.json + ecosystem.config.cjs
  ├── npm ci --omit=dev  (on server)
  │
  ├── mysqldump  →  /var/backups/tdd-todo-app-<timestamp>.sql
  ├── node dist/infrastructure/migrate.mjs  (on server)
  │
  └── pm2 reload tdd-todo-app  (or pm2 start on first deploy)
```

---

## Summary

The deploy pipeline went from zero to production-safe in a single session. The initial implementation got the ordering right (build → deploy → migrate → restart) and used several good defaults (`environment: production`, heredoc quoting for remote shell). A code review then caught four security/reliability gaps — TOFU host key trust, missing test gate, unguarded migration, and secret injection risk — all of which were fixed in the next two commits.

The end state is a pipeline that:
- Cannot deploy code that fails unit tests
- Cannot be manipulated by a compromised DNS/BGP path at deploy time
- Preserves a timestamped database snapshot before every schema change
- Keeps secret values out of shell command strings
- Runs `NODE_ENV=production` unconditionally via the pm2 config rather than relying on operator-supplied `.env` values
