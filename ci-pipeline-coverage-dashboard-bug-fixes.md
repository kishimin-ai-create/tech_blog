# Three Post-Deploy CI Bugs Fixed: Lock File, ESLint Crash, and Coverage Dashboard

**Type:** Devlog / Release Note  
**Audience:** Engineers maintaining or extending this project's CI pipeline  
**Commits:** `a04d96a` → `d1e40bc` → `7432af2`

---

## Background

After merging the GitHub Actions deploy workflow (previous session), the first CI run on `ubuntu-latest` exposed three separate failures that had gone unnoticed in local development on Windows. Each bug was independent — different root cause, different file, different fix — but all three had to be resolved before the pipeline became reliably green.

This post documents each bug: what failed, why it failed, and what was changed.

---

## Bug 1 — `npm ci` Fails on Linux: Missing esbuild Platform Entries

**Commit:** `d1e40bc`  
**File:** `backend/package-lock.json`

### Symptom

Both the deploy and test workflows failed immediately at the `npm ci` step:

```
npm error Missing: esbuild@0.28.0 from lock file
npm error Missing: @esbuild/linux-x64@0.28.0
npm error Missing: @esbuild/linux-arm64@0.28.0
...
```

### Root Cause

`backend/package-lock.json` had been generated on Windows. The lock file contained only Windows-specific optional platform packages for esbuild — `@esbuild/win32-x64`, `@esbuild/win32-ia32`, etc. — and none of the Linux equivalents.

`npm ci` is strict by design: it refuses to install if the lock file does not account for every expected optional dependency on the current platform. On `ubuntu-latest`, the Linux platform packages were expected but absent, causing a fatal error.

This is a common cross-platform trap. `npm install` only records optional packages that are *relevant to the current OS*, so a lock file generated on one OS is frequently incomplete for another.

### Fix

Re-ran `npm install` on the backend to regenerate the lock file with all-platform esbuild entries. The regenerated file gained 86 lines and dropped 19 (net +67), picking up `@esbuild/linux-x64`, `@esbuild/linux-arm64`, and other platform binaries alongside the existing Windows entries.

```
backend/package-lock.json | 105 +++++++++++++++++++++++++++----
 1 file changed, 86 insertions(+), 19 deletions(-)
```

### Takeaway

If your dev machine and CI runner use different operating systems, always regenerate `package-lock.json` on a Linux environment (or use `npm install --os=linux --cpu=x64` cross-compilation flags) before committing. Alternatively, run `npm ci` locally inside a Docker container that matches the CI image to catch mismatches early.

---

## Bug 2 — ESLint Crashes on `ecosystem.config.cjs`

**Commit:** `a04d96a`  
**File:** `backend/eslint.config.mts`

### Symptom

`npm run lint` in the backend failed with:

```
Error: Error while loading rule '@typescript-eslint/await-thenable':
You have used a rule which requires type information, but don't have parserOptions set.
Occurred while linting: backend/ecosystem.config.cjs
```

### Root Cause

`eslint.config.mts` applies `tseslint.configs.recommendedTypeChecked` at the top level — globally, with no file-pattern restriction:

```ts
// backend/eslint.config.mts (before fix)
tseslint.configs.recommended,
tseslint.configs.recommendedTypeChecked,   // ← applied to every file
```

`recommendedTypeChecked` includes rules such as `@typescript-eslint/await-thenable` that require the TypeScript compiler's type-checking API. The config does set up `parserOptions.projectService`, but that block is scoped to `**/*.{ts,mts,cts}` files only:

```ts
{
  files: ["**/*.{ts,mts,cts}"],
  languageOptions: {
    parserOptions: { projectService: true },
  },
},
```

`ecosystem.config.cjs` is a plain CommonJS file — it has no TypeScript parser configured. When ESLint's flat config processes it under the globally-applied type-aware rules, it has no type information to work with and crashes fatally.

### Fix

Added `"ecosystem.config.cjs"` to the `globalIgnores` entry so ESLint never processes it:

```diff
- ignores: ["coverage/**", "dist/**"],
+ ignores: ["coverage/**", "dist/**", "ecosystem.config.cjs"],
```

The diff is a single line. The result is that ESLint skips the file entirely rather than trying to apply type-aware rules to a file that has no TypeScript context.

### Takeaway

When using `tseslint.configs.recommendedTypeChecked`, scope it to TypeScript files only (`files: ["**/*.ts"]`) or add non-TypeScript config files to `globalIgnores`. Applying type-aware rules globally without restricting to TS-parseable files will crash on any plain JS/CJS file in the project.

---

## Bug 3 — Coverage Dashboard Always Shows 0%

**Commit:** `7432af2`  
**Files:** `scripts/generate-coverage-report.js`, `docs/coverage/index.html`

This commit fixed two distinct sub-bugs in the same script.

### Part A — Wrong JSON Structure Assumed (the 0% problem)

#### Symptom

The generated `docs/coverage/index.html` showed 0% for every metric (lines, branches, functions, statements) for every layer, even when tests were passing with real coverage data.

#### Root Cause

`extractMetrics()` read from `coverage.total`:

```js
// Before fix
const total = coverage.total || {};
return {
  lines:      Math.round(total.lines?.pct     || 0),
  branches:   Math.round(total.branches?.pct  || 0),
  functions:  Math.round(total.functions?.pct || 0),
  statements: Math.round(total.statements?.pct|| 0)
};
```

This assumes the input is `coverage-summary.json`, which *does* have a `total` key with pre-aggregated percentages. But Vitest/Istanbul actually produces `coverage-final.json` — a **file-coverage map**. In this format, every top-level key is an absolute file path; there is no `total`:

```json
{
  "/abs/path/to/src/app.ts": {
    "s": { "0": 5, "1": 3 },
    "b": { "0": [2, 1], "1": [0, 4] },
    "f": { "0": 4, "1": 0 }
  },
  "/abs/path/to/src/index.ts": { ... }
}
```

Because `coverage.total` is always `undefined`, every `?.pct` short-circuits to `undefined`, and `|| 0` silently falls back to zero. No error is ever thrown; the dashboard just renders all zeros.

#### Fix

`extractMetrics()` was rewritten to iterate all file entries and manually aggregate hit counts from the raw `s` (statements), `b` (branches), and `f` (functions) fields:

```js
function extractMetrics(coverage) {
  if (!coverage) return { lines: 0, branches: 0, functions: 0, statements: 0 };

  let stmtCovered = 0,   stmtTotal = 0;
  let branchCovered = 0, branchTotal = 0;
  let fnCovered = 0,     fnTotal = 0;

  for (const fileData of Object.values(coverage)) {
    // Statements: { "0": hitCount, "1": hitCount, ... }
    const stmts = fileData.s || {};
    for (const count of Object.values(stmts)) {
      stmtTotal++;
      if (count > 0) stmtCovered++;
    }

    // Branches: { "0": [taken, not_taken], "1": [...] }
    const branches = fileData.b || {};
    for (const counts of Object.values(branches)) {
      for (const count of counts) {
        branchTotal++;
        if (count > 0) branchCovered++;
      }
    }

    // Functions: { "0": hitCount, "1": hitCount, ... }
    const fns = fileData.f || {};
    for (const count of Object.values(fns)) {
      fnTotal++;
      if (count > 0) fnCovered++;
    }
  }

  const pct = (covered, total) =>
    total === 0 ? 0 : Math.round((covered / total) * 100);

  return {
    lines:      pct(stmtCovered, stmtTotal),   // Istanbul uses statements as proxy for lines
    branches:   pct(branchCovered, branchTotal),
    functions:  pct(fnCovered, fnTotal),
    statements: pct(stmtCovered, stmtTotal),
  };
}
```

Key aggregation rules:
- **Statements / Lines** — `s` is a map of statement IDs to hit counts; count entries with `count > 0`
- **Branches** — `b` is a map of branch IDs to arrays of arm counts (e.g., `[taken, not_taken]`); count individual arms with `count > 0`
- **Functions** — `f` is a map of function IDs to hit counts; count entries with `count > 0`

### Part B — "View Report" Links Led to 404

#### Symptom

Clicking any "View Report" link in the dashboard returned a 404 page.

#### Root Cause

The dashboard is written to `docs/coverage/index.html` — two directory levels below the repository root. The links used single-dot-dot relative paths:

```html
<!-- Before fix -->
<a href="../frontend/coverage/index.html">View Report</a>
<a href="../backend/coverage/unit/index.html">View Report</a>
```

From `docs/coverage/`, `../frontend/` resolves to `docs/frontend/coverage/index.html`, which does not exist. The actual report files live at `frontend/coverage/index.html` (repo root level), requiring two levels up.

#### Fix

Changed every report `href` from `../X/` to `../../X/`:

```diff
- <td><a href="../frontend/coverage/index.html" target="_blank">View Report</a></td>
+ <td><a href="../../frontend/coverage/index.html" target="_blank">View Report</a></td>

- <td><a href="../backend/coverage/unit/index.html" target="_blank">View Report</a></td>
+ <td><a href="../../backend/coverage/unit/index.html" target="_blank">View Report</a></td>
```

### Takeaway

Istanbul / Vitest produces two different JSON output files with different shapes:

| File | Top-level keys | Has `total`? | Use case |
|---|---|---|---|
| `coverage-summary.json` | `total`, `<file-path>`, … | ✅ Yes | Quick totals, simple reads |
| `coverage-final.json` | `<file-path>`, … (only) | ❌ No | Per-file detail, branch/statement data |

If you only need aggregate percentages, read `coverage-summary.json` (it has `total.lines.pct` etc.). If you consume `coverage-final.json`, you must aggregate `s`, `b`, and `f` yourself as shown above.

For relative-path links in generated HTML, always count the directory depth of the *output file*, not the script that generates it.

---

## Summary

| # | File Changed | Root Cause | Fix |
|---|---|---|---|
| 1 | `backend/package-lock.json` | Lock file generated on Windows; missing Linux esbuild platform packages | Regenerated with `npm install` to produce all-platform entries |
| 2 | `backend/eslint.config.mts` | `recommendedTypeChecked` applied globally; crashed on CJS file with no type parser | Added `ecosystem.config.cjs` to `globalIgnores` |
| 3a | `scripts/generate-coverage-report.js` | Read non-existent `coverage.total` from a file-map JSON; always 0% | Rewrote `extractMetrics` to aggregate `s`/`b`/`f` from all file entries |
| 3b | `scripts/generate-coverage-report.js` | Report `href` depth was 1 level short; linked to `docs/frontend/` instead of `frontend/` | Changed `../X/` → `../../X/` in all report hrefs |

All three bugs were invisible during local development on Windows and only surfaced on `ubuntu-latest` CI runners or when opening the dashboard in a browser — a reminder that cross-platform lock files, globally-scoped linting rules, and relative paths in generated output all deserve explicit verification before merging a CI workflow.
