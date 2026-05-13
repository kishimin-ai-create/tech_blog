# Fixing CI Lint Errors in UserProfilePage and User Profile Tests

## Overview

After implementing the user profile edit feature, the CI pipeline caught four distinct lint
violations across the frontend component, the backend test files, and a GitHub Actions workflow.
This article explains each error, its root cause, and the specific fix applied so the same
patterns can be avoided in future work.

**Affected files:**

| File | Rule(s) violated |
|---|---|
| `frontend/src/features/auth/pages/UserProfilePage.tsx` | `import/order`, `no-misused-promises` |
| `backend/src/infrastructure/user-profile-validation.small.test.ts` | `vitest/no-conditional-expect`, `stylistic/semi` |
| `backend/src/tests/user-profile.medium.test.ts` | `stylistic/semi` |
| `.github/workflows/ci-nightly.yml` | Step skipped when earlier steps fail |

---

## Fix 1 — `import/order`: Wrong Import Ordering in UserProfilePage.tsx

### Error

```
import/order: 'jotai' import must come before 'react'
```

### Cause

The project's ESLint config (via `eslint-plugin-import`) enforces that external package imports
are sorted alphabetically. The original code listed `react` before `jotai`:

```ts
// ❌ Before
import { useState } from 'react'
import { useAtom } from 'jotai'
```

Since `j` comes before `r`, `jotai` must appear first.

### Fix

Swap the two lines:

```ts
// ✅ After
import { useAtom } from 'jotai'
import { useState } from 'react'
```

No runtime behaviour changes — this is a purely cosmetic rule that keeps imports scannable and
consistent across the codebase.

---

## Fix 2 — `no-misused-promises`: Async Function Passed Directly to `onSubmit`

### Error

```
no-misused-promises: Promise-returning function provided to attribute where a void return was expected
```

### Cause

`handleSubmit` is an `async` function, which means it returns a `Promise`. React's `onSubmit`
prop expects a handler that returns `void`. Assigning an async function directly violates this
contract:

```tsx
// ❌ Before — handleSubmit returns Promise<void>, not void
<form onSubmit={handleSubmit}>
```

If the `Promise` were to reject unexpectedly, the rejection would be silently swallowed by React,
and the TypeScript/ESLint rule `@typescript-eslint/no-misused-promises` is designed to surface
exactly this class of bug.

### Fix

Wrap the async call in a synchronous lambda and chain `.catch()` to make error handling explicit:

```tsx
// ✅ After
<form onSubmit={(e) => { handleSubmit(e).catch(() => { /* handled inside */ }) }}>
```

The comment `/* handled inside */` signals that error handling already lives inside `handleSubmit`
itself (it has a `try/catch` block that calls `setError`). The `.catch()` wrapper is there purely
to satisfy the `void` return requirement and to prevent unhandled-rejection warnings.

---

## Fix 3 — `vitest/no-conditional-expect`: Guards Around `expect()` in Tests

### Error

```
vitest/no-conditional-expect: Expect must not be called in a conditional
```

### Cause

Several tests in `user-profile-validation.small.test.ts` used an `if` guard before accessing
discriminated union members:

```ts
// ❌ Before — expect() is inside an if-block
const result = parseUpdateUserProfileInput(body)
if (result.success) {
  expect(result.email).toBe('valid@example.com')
}
```

The problem here is twofold:

1. **The rule is violated**: `vitest/no-conditional-expect` forbids `expect()` calls inside
   conditionals because if the condition is `false`, the assertion is silently skipped rather than
   failing the test. The test would pass even when `result.success` is `false`.
2. **TypeScript narrowing is lost**: After the `if` block, `result` is no longer narrowed to the
   success branch, so properties like `result.email` are not type-safe outside it.

This pattern appeared 6 times across the file, covering cases for `email`, `currentPassword`, and
`newPassword` field assertions.

### Fix

Import `assert` from Vitest and call it unconditionally before the `expect()`:

```ts
// ✅ After — assert() throws if false, so expect() is always reached
import { assert, describe, expect, it } from 'vitest'

const result = parseUpdateUserProfileInput(body)
assert(result.success)               // throws (fails the test) if false
expect(result.email).toBe('valid@example.com')  // now type-safe and always executed
```

`assert(result.success)` acts as a type guard: if it passes, TypeScript knows `result` is in the
success branch for every line that follows. If it fails, the test fails immediately with a clear
message instead of silently passing.

All 6 occurrences of the `if (result.success) { expect(...) }` pattern were replaced with this
two-line form.

---

## Fix 4 — `stylistic/semi`: Missing Semicolons in Backend Test Files

### Error

```
stylistic/semi: Missing semicolon
```

### Cause

The backend ESLint configuration enforces trailing semicolons (`stylistic/semi`). Both
`user-profile-validation.small.test.ts` and `user-profile.medium.test.ts` were written without
semicolons at the end of statements, which was consistent with the style used during the Red
(failing tests) and Green (implementation) phases but diverged from the enforced rule.

### Fix

Run ESLint's auto-fixer:

```bash
npx eslint --fix backend/src/infrastructure/user-profile-validation.small.test.ts
npx eslint --fix backend/src/tests/user-profile.medium.test.ts
```

This added semicolons to every statement in both files in a single pass. No logic was changed.

---

## Fix 5 — CI Nightly: Large E2E Step Skipped on Earlier Failure

### Problem

In `.github/workflows/ci-nightly.yml`, the "Run large E2E tests" step had no `if:` condition.
GitHub Actions skips subsequent steps when a prior step fails, so if the "Run medium E2E tests"
step failed, the large E2E tests were never executed at all — hiding potential failures in that
suite.

```yaml
# ❌ Before — step is skipped if medium E2E step fails
- name: Run large E2E tests
  run: npm run test:e2e:large
```

### Fix

Add `if: always()` to force the step to run regardless of the outcome of preceding steps:

```yaml
# ✅ After — step always runs
- name: Run large E2E tests
  if: always()
  run: npm run test:e2e:large
  env:
    RUN_VISUAL_TESTS: 1
    PLAYWRIGHT_BASE_URL: http://localhost:4173
```

Note that the "Upload Playwright report" step already had `if: always()`, so reports were
uploaded even on failure. This fix brings the large E2E step in line with the same intent.

---

## Verification

After applying all fixes:

```bash
cd frontend && npm run lint       # 0 errors
cd backend  && npm run lint       # 0 errors
cd frontend && npm run typecheck  # 0 errors
cd backend  && npm run typecheck  # 0 errors
```

---

## Summary

| # | Rule | File | Fix |
|---|---|---|---|
| 1 | `import/order` | `UserProfilePage.tsx` | Reordered imports alphabetically (`jotai` before `react`) |
| 2 | `no-misused-promises` | `UserProfilePage.tsx` | Wrapped async handler with `.catch()` lambda |
| 3 | `vitest/no-conditional-expect` | `user-profile-validation.small.test.ts` | Replaced `if` guards with `assert()` + unconditional `expect()` |
| 4 | `stylistic/semi` | Both backend test files | ESLint auto-fix (`--fix`) added missing semicolons |
| 5 | CI step dependency | `ci-nightly.yml` | Added `if: always()` to large E2E step |

### Key takeaways

- **`no-misused-promises`** is a safety rule, not cosmetic. Passing an `async` function directly
  to a `void`-typed event prop can hide unhandled rejections. Always wrap with a synchronous
  lambda.
- **`vitest/no-conditional-expect`** prevents silent test passes. Use `assert()` from Vitest as a
  type-narrowing guard before accessing discriminated union members in tests.
- **`if: always()` in CI** ensures that independent test suites are always exercised, giving a
  complete picture of the build health even when earlier steps fail.
