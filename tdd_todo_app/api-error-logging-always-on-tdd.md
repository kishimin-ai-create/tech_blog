# API Error Logging Always-On: A Complete TDD Cycle

## Target Readers

Backend engineers who want to see a concrete, end-to-end TDD cycle on a small but
production-critical feature — from spotting the bug, writing failing tests, applying
the minimal fix, refactoring, and responding to code review.

---

## Background: The Bug That Silenced Errors

The `backend/src/infrastructure/hono-app.ts` file already had working API logging
middleware before this cycle. Two functions handled the output:

- `logSuccessRequest()` — printed `[METHOD] path → status (Xms)` for 2xx responses
- `logErrorRequest()` — printed `[METHOD] path → ERROR status — code: message` for 4xx/5xx responses

Both functions started with the same guard:

```typescript
if (process.env.LOG_API_REQUESTS !== 'true') {
  return;
}
```

The guard made sense for `logSuccessRequest()`. High-traffic production deployments
don't need every 200 logged; operators opt in with `LOG_API_REQUESTS=true`.

But `logErrorRequest()` had **exactly the same guard**. The consequence: in any
environment where `LOG_API_REQUESTS` was absent (production default, CI, most local
runs), **all error responses were silently swallowed**. A 401, a 409, a 500 — nothing
appeared in the logs. The feature spec in `docs/spec/features/api-logging.md` was
explicit:

> Error responses (4xx, 5xx) are **always logged** regardless of the
> `LOG_API_REQUESTS` environment variable. This ensures that errors are never silently
> swallowed in any environment.

The implementation did not match the spec. The fix was one deleted block; the TDD
cycle documented exactly why.

---

## Red Phase: Writing Tests That Expose the Contract

A new `describe` block was added to
`backend/src/tests/integrations/infrastructure/hono-app.medium.test.ts`:

```typescript
describe('Error logging always-on behavior (no LOG_API_REQUESTS env var)', () => {
  let consoleLogSpy: ReturnType<typeof vi.spyOn>;

  beforeEach(() => {
    // Guarantee the var is absent — this is the exact production default
    delete process.env.LOG_API_REQUESTS;
    consoleLogSpy = vi.spyOn(console, 'log').mockImplementation(() => {});
  });

  afterEach(() => {
    consoleLogSpy.mockRestore();
    delete process.env.LOG_API_REQUESTS;
  });
  // ...
});
```

The `beforeEach` calls `delete process.env.LOG_API_REQUESTS` rather than setting it
to `undefined` or `'false'`. This precisely mirrors production: the variable is
absent, not set to any value. Every test in this block runs without the env var.

Five test cases were written:

| # | Test | What it verifies |
|---|------|-----------------|
| 1 | `should log 4xx error response even when LOG_API_REQUESTS is not set` | A `POST /api/v1/apps` with an empty body produces a 422 **and** a matching log entry |
| 2 | `should log 401 error response even when LOG_API_REQUESTS is not set` | A login with non-existent credentials logs `[POST] … → ERROR 401` |
| 3 | `should log 409 conflict error even when LOG_API_REQUESTS is not set` | Duplicate signup logs `→ ERROR 409` and contains `EMAIL_ALREADY_EXISTS` |
| 4 | `should NOT log successful 2xx response when LOG_API_REQUESTS is not set` | `GET /api/v1/apps → 200` produces **no** log entry |
| 5 | `should log 4xx error but NOT log 2xx success when LOG_API_REQUESTS is not set` | Mixed scenario: 422 is logged, 200 is not |

Test 3 is worth examining in detail because it required a two-step setup: sign up
once to create the account, then attempt a duplicate signup:

```typescript
it('should log 409 conflict error even when LOG_API_REQUESTS is not set', async () => {
  const { app } = buildApp();

  // First signup (201) — success logs are suppressed without LOG_API_REQUESTS
  await req(app, 'POST', '/api/v1/auth/signup', {
    email: 'always-on@example.com',
    password: 'password123',
  });
  // 201 success is not logged without LOG_API_REQUESTS — no mockClear() needed here

  // Duplicate signup → 409
  const res = await req(app, 'POST', '/api/v1/auth/signup', {
    email: 'always-on@example.com',
    password: 'password123',
  });
  expect(res.status).toBe(409);

  // Assert both the presence and the format
  const errorLogCall = consoleLogSpy.mock.calls.find((call: unknown[]) =>
    typeof call[0] === 'string' &&
    call[0].includes('ERROR') &&
    call[0].includes('409')
  );
  expect(errorLogCall).toBeDefined();
  const logMessage = errorLogCall![0] as string;
  expect(logMessage).toMatch(/→ ERROR 409/);
  expect(logMessage).toContain('EMAIL_ALREADY_EXISTS');
});
```

Before the fix, all five tests in this block would fail: `consoleLogSpy` would
capture zero calls because `logErrorRequest()` returned immediately at the guard.

---

## Green Phase: The One-Line Fix

With the failing tests committed, the fix was straightforward. The guard block was
removed from `logErrorRequest()`:

**Before:**
```typescript
async function logErrorRequest(
  method: string, path: string, status: number, context: Context
): Promise<void> {
  if (process.env.LOG_API_REQUESTS !== 'true') {  // ← guarded
    return;
  }
  const errorDetails = await extractErrorDetails(context);
  // ...
}
```

**After:**
```typescript
async function logErrorRequest(
  method: string, path: string, status: number, context: Context
): Promise<void> {
  // No guard — always logs regardless of LOG_API_REQUESTS
  const errorDetails = await extractErrorDetails(context);
  // ...
}
```

The JSDoc was updated in the same commit to reflect the new contract:

```typescript
/**
 * Logs an error API request with format: [METHOD] path → ERROR status — code: message
 * If error details cannot be extracted, falls back to: [METHOD] path → ERROR status
 * Always logs regardless of LOG_API_REQUESTS environment variable.
 */
```

`logSuccessRequest()` was left untouched — its guard is intentional and correct.

The resulting behavior, aligned with the spec:

| Response type | Status range | `LOG_API_REQUESTS=true` | `LOG_API_REQUESTS` absent |
|---|---|---|---|
| Success | 2xx | Logged | **Silent** |
| Error | 4xx, 5xx | Logged | **Logged** |

All 342 integration tests passed after the removal.

---

## Refactor Phase: Clarity Before Moving On

Two independent cleanups followed once green.

### Guard clauses in `extractErrorDetails()`

The function previously used nested if/else blocks to validate the response body
structure:

```typescript
// Before — nested
if (typeof responseBody === 'object' && responseBody !== null && 'error' in responseBody) {
  const errorObj = responseBody.error;
  if (typeof errorObj === 'object' && /* ... more checks ... */) {
    return { code: errorObj.code, message: errorObj.message };
  } else {
    console.warn('[logging] Error object has unexpected structure …');
  }
} else {
  console.warn('[logging] Response body missing "error" property');
}
```

The refactor flipped each condition into a guard clause (early return on failure),
applying De Morgan's laws to invert the checks:

```typescript
// After — guard clauses
if (typeof responseBody !== 'object' || responseBody === null || !('error' in responseBody)) {
  console.warn('[logging] Response body missing "error" property');
  return null;
}

const errorObj = responseBody.error;

if (
  typeof errorObj !== 'object' ||
  errorObj === null ||
  !('code' in errorObj) ||
  !('message' in errorObj) ||
  typeof errorObj.code !== 'string' ||
  typeof errorObj.message !== 'string'
) {
  console.warn('[logging] Error object has unexpected structure or non-string code/message');
  return null;
}

return { code: errorObj.code, message: errorObj.message };
```

The happy-path `return` is now at the bottom, reached only after every guard has
passed. The nesting level dropped by one level and the logic flows in a single
direction. The behavior is identical — the change is purely structural.

### Variable rename: `elapsedTime` → `elapsedTimeMs`

Inside the middleware's `afterEach` handler, the elapsed time variable was renamed:

```typescript
// Before
const elapsedTime = Date.now() - startTime;
logSuccessRequest(method, path, status, elapsedTime);

// After
const elapsedTimeMs = Date.now() - startTime;
logSuccessRequest(method, path, status, elapsedTimeMs);
```

The suffix `Ms` makes the unit explicit at the declaration site, which matters when
reading the log output `(${elapsedTimeMs}ms)` — the unit appears twice and they
agree. A small change, but it removes ambiguity for the next engineer reading the
code.

---

## Review Phase: Three Low-Priority Findings Fixed

A code review on the new test block (`review/api-logging-20250725.md`) raised three
P3 findings. All were fixed in a single follow-up commit.

### Finding 1 — Test 1 only checked presence, not format

The original assertion for the 422 always-on test looked for the strings `'ERROR'`
and `'422'` anywhere in the log message. A hypothetical string like
`"ERROR: status 422 happened"` would satisfy that check even though it does not
match the spec format `[METHOD] path → ERROR status — code: message`.

**Fix:** added a regex assertion after the presence check, consistent with how
tests 2–5 were already written:

```typescript
expect(errorLogCall).toBeDefined();
const logMessage = errorLogCall![0] as string;
expect(logMessage).toMatch(/\[POST\].*→ ERROR 422/);  // ← added
```

### Finding 2 — `console.warn` not suppressed in the new block

`extractErrorDetails()` calls `console.warn()` on every extraction failure path
(malformed JSON, missing `error` property, wrong structure, size guard exceeded).
The existing `beforeEach` only mocked `console.log`, leaving warn calls free to
leak into CI output.

**Fix:** added a `consoleWarnSpy` alongside `consoleLogSpy`:

```typescript
let consoleWarnSpy: ReturnType<typeof vi.spyOn>;

beforeEach(() => {
  delete process.env.LOG_API_REQUESTS;
  consoleLogSpy = vi.spyOn(console, 'log').mockImplementation(() => {});
  consoleWarnSpy = vi.spyOn(console, 'warn').mockImplementation(() => {});
});

afterEach(() => {
  consoleLogSpy.mockRestore();
  consoleWarnSpy.mockRestore();
  delete process.env.LOG_API_REQUESTS;
});
```

### Finding 3 — `mockClear()` in the 409 test was a silent no-op

The 409 test signed up once (201) before triggering the duplicate 409. After the
first signup, the original code called `consoleLogSpy.mockClear()`. But because
`LOG_API_REQUESTS` is absent throughout this block, `logSuccessRequest()` returns
early and never calls `console.log` — meaning the spy holds zero calls at that
point and `mockClear()` clears nothing.

**Fix:** removed the call, replaced with a comment explaining why it is unnecessary:

```typescript
// 201 success is not logged without LOG_API_REQUESTS — no mockClear() needed here
```

This makes the intent explicit for future readers who might wonder whether the spy
had captured something meaningful before the clear.

All 56 tests in the file continued to pass after each of these fixes.

---

## Lessons from This Cycle

**1. Symmetric guards are not always correct.**  
Copying a guard from one function to a sibling is tempting because it looks
consistent. But `logSuccessRequest()` and `logErrorRequest()` have fundamentally
different observability contracts. Success logs are optional noise; error logs are
mandatory signal. The symmetry was a bug hiding as consistency.

**2. A `beforeEach` that deletes an env var is stronger than one that sets it to a falsy value.**  
`delete process.env.LOG_API_REQUESTS` exactly mirrors "not deployed with this
variable". `process.env.LOG_API_REQUESTS = undefined` or `= 'false'` do not —
`'false' !== 'true'` still returns `true` for the early-return check, but the
intent is muddied. Use `delete` for "variable absent" semantics.

**3. Guard clauses reduce the cognitive load of validating complex shapes.**  
The `extractErrorDetails()` refactor is a textbook example: each guard eliminates
one bad state and returns early. By the time execution reaches the bottom, all
invariants are proven. The happy path reads like a straight line.

**4. `mockClear()` on an empty spy is not just harmless — it is misleading.**  
When a spy is cleared and the reader cannot tell whether it had captured calls, they
have to reason about the entire prior execution to understand whether the clear
mattered. A one-line comment does that reasoning for them.

**5. Presence checks and format checks serve different purposes.**  
Test 1 after the review fix has two assertions: `expect(errorLogCall).toBeDefined()`
verifies the log was emitted at all; `expect(logMessage).toMatch(/\[POST\].*→ ERROR 422/)` 
verifies it matches the spec format. A test that only checks presence will survive a
regression that changes the format. Keep both.

---

## Summary

| Phase | Commit | Change |
|-------|--------|--------|
| Red | `ed5da5a` | 5 new tests: `describe('Error logging always-on behavior …')` |
| Green | `ed5da5a` | Removed `LOG_API_REQUESTS` guard from `logErrorRequest()` |
| Refactor | `3f13b58` | Flattened nested if/else in `extractErrorDetails()` to guard clauses |
| Refactor | `103a59c` | Renamed `elapsedTime` → `elapsedTimeMs` |
| Review docs | `be0b542` | Added review findings doc (`review/api-logging-20250725.md`) |
| Review fix | `a59aea0` | Format assertion in test 1; `consoleWarnSpy`; removed no-op `mockClear()` |

The root bug was a single misapplied guard: one `if` block that should never have
been in `logErrorRequest()`. The TDD cycle turned that gap into five executable
tests, fixed it in one deletion, improved the surrounding code's readability, and
caught three subtle test-quality issues through review — leaving the codebase
clearer than it started.
