# Fix: Signup Always Showed "Sign up failed" Instead of the Real Error Message

**Date:** 2026-05-11  
**Tech stack:** React · TypeScript · Hono (Node.js) · Render.com · MSW · Vitest  
**Changed files:**
- `frontend/src/features/auth/hooks/useAuthForm.ts`
- `frontend/src/features/auth/pages/SignupPage.medium.test.tsx`

---

## Error Overview

When a user tried to sign up with invalid data in production, the UI always displayed
the hardcoded fallback message **"Sign up failed"** — regardless of what the backend
actually reported.

The expected behavior was to show a specific, actionable message such as
**"Password must be at least 8 characters."** Instead, every validation failure
silently collapsed into the same generic string, giving users no indication of what
to fix.

---

## Cause

### The backend's error shape

The Hono backend returns the following JSON body on a 422 Validation Error:

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Password must be at least 8 characters."
  }
}
```

`error` is an **object**, not a string.

### The proxy complication on Render.com

The frontend is deployed as a Render.com static site that proxies `/api/*` to the
backend web service. Under certain conditions, Render's proxy layer converts a
backend `4xx` response into a `200` (or another `2xx`). This means `response.ok`
evaluates to `true`, even though the body carries `{ success: false, error: { ... } }`.

### The broken guard in `parseAuthResponse`

`useAuthForm.ts` contained a pure parser function, `parseAuthResponse`, that decoded
the JSON body once `response.ok` was confirmed to be `true`. The relevant section
**before the fix** read:

```typescript
// ❌ Before fix (lines 67-68 in original)
if (typeof value.error !== 'string') return null

return { success: false, error: value.error }
```

The intent was: "if there is no string error, bail out." In practice, when the proxy
converted a 422 to a 200, the body arrived with `value.error` set to an object —
the `typeof value.error !== 'string'` guard was `true`, and the function returned
`null` immediately.

### The fallback masks the real message

Back in the caller (`submitAuthRequest`), a `null` result from `parseAuthResponse`
fell through directly to:

```typescript
if (!authResponse) {
  setError(fallbackErrorMessage)   // "Sign up failed" — always
  return
}
```

So the actual validation message was discarded at the parser boundary, replaced by
the opaque fallback, and never reached the UI.

---

## Fix

The fix extended `parseAuthResponse` to handle **both** error shapes — a plain
string and an object with a `message` field — instead of treating anything non-string
as an unrecoverable parse failure.

**After the fix** (`useAuthForm.ts`):

```typescript
// ✅ After fix
if (typeof value.error === 'string') {
  return { success: false, error: value.error }
}

if (isRecord(value.error) && typeof value.error.message === 'string') {
  return { success: false, error: value.error.message }
}

return null
```

The helper `isRecord` (already present in the file) checks that a value is a
non-null object:

```typescript
const isRecord = (value: unknown): value is Record<string, unknown> =>
  typeof value === 'object' && value !== null
```

With this change:

| `value.error` shape | Behaviour |
|---|---|
| `"some string"` | Extracted directly — unchanged from before |
| `{ code: "...", message: "..." }` | `message` is extracted — **newly supported** |
| Anything else | Returns `null` (fallback shown) — unchanged |

The `null` path is still reachable, but only for genuinely malformed bodies, not for
the backend's documented validation error structure.

---

## Test Coverage (TDD Cycle)

The fix followed a strict RED → GREEN cycle.

### RED — failing test first

A new medium test was added to `SignupPage.medium.test.tsx` to reproduce the exact
production scenario. MSW intercepts the signup request and returns a `200` with an
object-style error body:

```typescript
it('when signup API returns 200 with object-style error body, then error message from body is displayed', async () => {
  // Arrange — simulates a proxy that converts backend 4xx to 200, preserving the JSON body
  server.use(
    http.post('/api/v1/auth/signup', () =>
      HttpResponse.json(
        {
          success: false,
          error: { code: 'VALIDATION_ERROR', message: 'Password must be at least 8 characters.' },
        },
        { status: 200 },
      ),
    ),
  )
  renderWithProviders(<SignupPage />)

  await user.type(screen.getByRole('textbox', { name: /email/i }), 'test@example.com')
  await user.type(screen.getByLabelText(/password/i), 'password123')
  await user.click(screen.getByRole('button', { name: /sign up/i }))

  // Assert
  expect(await screen.findByRole('alert')).toHaveTextContent('Password must be at least 8 characters.')
})
```

Before the fix, this test failed: the alert contained `"Sign up failed"` instead of
the expected message.

### Additional coverage: 201 status happy path

A second test was added (placed in the `Error Handling` describe block, but
testing the success boundary) to ensure `parseAuthResponse` correctly handles the
backend's `201 Created` response — which is also a `2xx` and therefore goes through
the same `response.ok` branch:

```typescript
it('when signup succeeds with 201 status, then authAtom is populated and page navigates to app-list', async () => {
  server.use(
    http.post('/api/v1/auth/signup', () =>
      HttpResponse.json(
        {
          success: true,
          data: { token: 'test-token', user: { id: 'user-1', email: 'test@example.com' } },
        },
        { status: 201 },
      ),
    ),
  )
  // ... assert authAtom and currentPageAtom updated correctly
})
```

### GREEN — all tests passing

After the `parseAuthResponse` change, both new tests and all 100+ existing tests
passed:

```
npm run lint && npm run typecheck && npm run test:small && npm run test:medium
# → 102 tests, all pass
```

---

## Summary

| | Detail |
|---|---|
| **Symptom** | Every signup failure showed `"Sign up failed"` regardless of the backend's reason |
| **Root cause** | `parseAuthResponse` used `typeof value.error !== 'string'` as a guard, so it returned `null` for object-style errors from Hono, and the caller fell through to the hardcoded fallback |
| **Triggering condition** | Render.com proxy converting backend `422` to `200`, so `response.ok === true` and the object-style body reached `parseAuthResponse` |
| **Fix** | Added an explicit branch for `{ code, message }` shaped errors; `value.error.message` is now extracted correctly |
| **Lines changed** | 4 lines in `useAuthForm.ts` (`-2 / +6`) |
| **Test added** | 2 new medium tests — one for the bug scenario, one for the `201` happy path |

The core lesson is a familiar one: **a `null` return from a parser is a silent
discard.** When a parser is the single choke-point between a raw network body and the
UI, an overly narrow type guard can suppress legitimate data just as effectively as a
missing catch block. The fix keeps the fallback path for truly malformed responses,
while ensuring the backend's documented error structure is handled explicitly.
