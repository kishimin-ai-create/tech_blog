# Implementing User Profile Edit with TDD

**Date:** 2026-05-11  
**Target readers:** Full-stack engineers familiar with React, TypeScript, and Hono; developers looking to apply TDD discipline across the full stack  
**Scope:** This article covers the complete Red → Green TDD cycle for the user profile edit feature: a `PUT /api/v1/users/:userId` backend endpoint, the `UserProfilePage` frontend component, and the Playwright E2E tests that close the loop. It does not cover actual persistence of the updated email or password to a database (the current implementation echoes values back without storage).

---

## Background

The TDD todo app had a working authentication flow — signup, login, logout — all backed by Jotai's `authAtom` (`atomWithStorage`). What was still missing was a way for a logged-in user to change their own email or password without going through the entire signup flow again.

The goal: add a profile edit screen behind a "プロフィール" button in the header, drive the full stack by tests first, and keep every layer — validation logic, HTTP handler, React component, and browser flow — covered before a single production file was created.

---

## The TDD Cycle

### Red Phase — Write All Tests First

Two commits form the Red phase: `f36df6a` (test) and `e0ae283` (green). All 56 tests were written in a single commit while every production file was absent. The five test files, grouped by layer:

| File | Tier | Tests | Initial result |
|---|---|---|---|
| `backend/src/infrastructure/user-profile-validation.small.test.ts` | Small (unit) | 19 | All FAIL — module not found |
| `backend/src/tests/user-profile.medium.test.ts` | Medium (integration) | 16 | All FAIL — route returns 404 |
| `frontend/src/shared/navigation.small.test.ts` | Small (unit) | 4 | All FAIL — `goToUserProfile` undefined |
| `frontend/src/features/auth/pages/UserProfilePage.medium.test.tsx` | Medium (integration) | 12 | All FAIL — module not found |
| `frontend/e2e/user-profile.medium.test.ts` | Medium (Playwright E2E) | 5 | All FAIL — element not found |

Fifty-six failing tests. No production code. That is the intended starting state.

The commit message for the Red phase doubles as living documentation:

```
- backend/src/infrastructure/user-profile-validation.small.test.ts
  Unit tests for the not-yet-existing parseUpdateUserProfileInput function:
  valid email, both passwords, invalid email formats, password-pair
  constraints, and boundary email values (19 tests, all FAIL)

- backend/src/tests/user-profile.medium.test.ts
  Integration tests for the not-yet-existing PUT /api/v1/users/:userId route:
  200 with valid email, 200 with both passwords, 422 for invalid email,
  422 for mismatched password pairs (16 tests, all FAIL - route returns 404)
```

The failing tests define the exact API contract before any implementation decision is made.

---

### Green Phase — Minimum Implementation to Pass

`e0ae283` adds exactly the production code required to turn all 56 tests green. Six files changed, 217 lines added. No production code beyond what the tests demanded.

---

## Backend: `PUT /api/v1/users/:userId`

### Validation — `parseUpdateUserProfileInput`

The first new file is `backend/src/infrastructure/user-profile-validation.ts`. Its single exported function, `parseUpdateUserProfileInput`, validates the raw request body and returns a typed discriminated union:

```ts
export function parseUpdateUserProfileInput(body: unknown):
  | { success: true; email: string; currentPassword?: string; newPassword?: string }
  | { success: false; message: string }
```

The validation rules encoded by the 19 unit tests:

| Rule | Behaviour |
|---|---|
| `email` missing or not a string | `{ success: false, message: 'Email is required.' }` |
| Email fails regex `^[^\s@]+@[^\s@]+\.[^\s@]+$` | `{ success: false, message: 'Please enter a valid email address.' }` |
| Email has leading/trailing whitespace | Trimmed before validation and returned |
| `newPassword` present but `currentPassword` absent | `{ success: false, message: 'currentPassword is required ...' }` |
| `currentPassword` present but `newPassword` absent | `{ success: false, message: 'newPassword is required ...' }` |
| Either password is an empty string | `{ success: false, message: 'Passwords cannot be empty.' }` |
| Valid email only | `{ success: true, email }` |
| Valid email + both passwords | `{ success: true, email, currentPassword, newPassword }` |

Password update is **always a pair**: both fields must be provided together or neither. Passing only one is a validation error. This rule was captured by specific unit tests before the function existed:

```ts
it('when body has newPassword but no currentPassword, then returns success false', () => {
  const result = parseUpdateUserProfileInput({
    email: 'valid@example.com',
    newPassword: 'new-pass',
  })
  expect(result.success).toBe(false)
})
```

### HTTP Handler — `hono-app.ts`

The route is added to the Hono app with minimal ceremony:

```ts
app.put('/api/v1/users/:userId', async c => {
  const parsed = parseUpdateUserProfileInput(await readRequestBody(c))
  const userId = c.req.param('userId')

  if (!parsed.success) {
    return c.json(
      {
        success: false,
        data: null,
        error: { code: 'VALIDATION_ERROR', message: parsed.message },
      },
      422,
    )
  }

  return c.json(
    {
      success: true,
      data: { id: userId, email: parsed.email },
    },
    200,
  )
})
```

The handler itself contains no validation logic — it delegates entirely to `parseUpdateUserProfileInput`. The integration tests verified this structure through 16 scenarios covering status codes, response body shape, `Content-Type` header, and the full range of validation error conditions, all using the real Hono app instance:

```ts
it('when called with a valid email, then response data contains the updated email', async () => {
  const res = await request('PUT', `/api/v1/users/user-1`, {
    email: 'updated@example.com',
  })
  const json = await res.json()
  expect(json.data.email).toBe('updated@example.com')
})

it('when called with mismatched password pair (newPassword only), then returns 422', async () => {
  const res = await request('PUT', `/api/v1/users/user-1`, {
    email: 'valid@example.com',
    newPassword: 'secret',
  })
  expect(res.status).toBe(422)
})
```

---

## Frontend: `UserProfilePage`

### Navigation — `useNavigation` Extension

Before building the page component, the navigation layer needed a new entry. The `Page` union type and the `useNavigation` hook in `frontend/src/shared/navigation.ts` were extended together:

```ts
// navigation.ts
export type Page =
  | { name: 'landing' }
  | { name: 'login' }
  | { name: 'signup' }
  | { name: 'app-list' }
  | { name: 'app-detail'; appId: string }
  | { name: 'app-create' }
  | { name: 'app-edit'; appId: string }
  | { name: 'user-profile' }        // ← new

export function useNavigation() {
  // ...
  return {
    // existing navigators...
    goToUserProfile: () => setPage({ name: 'user-profile' }),   // ← new
  }
}
```

Four small unit tests guarded this change: one confirms that `goToUserProfile` is a function on the returned object, and three verify that calling it sets `currentPageAtom` to `{ name: 'user-profile' }` from different starting pages. All four were written before the code.

### The `UserProfilePage` Component

`frontend/src/features/auth/pages/UserProfilePage.tsx` is 128 lines. Its responsibilities:

1. **Pre-fill email** from `authAtom` so the user sees their current address immediately.
2. **Conditionally render** the "現在のパスワード" field — it only appears when the user has started typing into "New Password", avoiding unnecessary UI noise.
3. **Update `authAtom`** with the new email on a successful `200` response, so the rest of the app sees the updated address without a page reload.
4. **Display inline feedback** — a success message ("保存しました") or a server-provided error message, both with accessible role markup.
5. **Back navigation** — a "← 戻る" button calls `goToAppList()`, returning the user to the app-list screen.

The key state-update block after a successful API response:

```tsx
if (json.success && json.data) {
  setAuth({
    token: auth!.token,
    user: { id: auth!.user.id, email: json.data.email },
  })
  setSuccessMessage('保存しました')
}
```

Only the `email` field is updated; the existing `token` and `user.id` are preserved. The `useAtom(authAtom)` hook causes `atomWithStorage` to write the new value to `localStorage`, so the updated email persists across page reloads.

The medium integration tests for the component cover 12 scenarios using MSW (`http.put`) to stub the API:

```ts
describe('Content - Initial Form State', () => {
  it('when rendered with a logged-in user, then the email input is pre-filled with the current user email', () => {
    const store = createStore()
    store.set(authAtom, MOCK_AUTH)
    renderWithProviders(<UserProfilePage />, { store })
    expect(screen.getByRole('textbox', { name: /email/i })).toHaveValue('current@example.com')
  })
  // ...
})

describe('Happy Path - Successful Profile Update (email only)', () => {
  it('when a new email is submitted and the API responds with 200, then "保存しました" success message is displayed', async () => {
    server.use(
      http.put('/api/v1/users/:userId', () =>
        HttpResponse.json({ success: true, data: { id: 'user-1', email: 'new@example.com' } }),
      ),
    )
    // ...
    await user.click(screen.getByRole('button', { name: /保存/i }))
    await waitFor(() => expect(screen.getByText('保存しました')).toBeInTheDocument())
  })
})
```

### App Routing — `App.tsx`

Two changes in `App.tsx` wire everything together:

1. A "プロフィール" button is rendered in the authenticated header, wired to `goToUserProfile()`.
2. A new conditional branch renders `<UserProfilePage />` when `currentPage.name === 'user-profile'`, inserted *before* the default authenticated layout so it takes over the full viewport:

```tsx
const { goToUserProfile } = useNavigation()

// In the authenticated branch:
if (currentPage.name === 'user-profile') {
  return <UserProfilePage />
}

return (
  <div>
    <LogoutButton />
    <button type="button" onClick={goToUserProfile}>
      プロフィール
    </button>
    <AppListPage />
    {/* ... */}
  </div>
)
```

---

## Playwright E2E Tests

Five E2E tests in `frontend/e2e/user-profile.medium.test.ts` close the outer loop. They go through the real browser flow — signup form → app list → profile edit — without touching a real backend:

| Test | What it verifies |
|---|---|
| Profile button visible in header after login | The "プロフィール" button is in the DOM after signup |
| Clicking profile button shows the edit page | Navigation from header to profile edit works |
| Email input pre-filled with current user email | `authAtom` value is correctly read by the component |
| Saving a new email shows "保存しました" | Full form submit → API stub → success feedback loop |
| "← 戻る" button returns to app list | Back navigation leaves the profile page |

The tests use the same `page.route()` stub pattern established in the logout E2E tests. A signup stub, an app-list stub, and a profile-update stub are composed per test:

```ts
async function registerUpdateUserProfileSuccessStub(page: Page, updatedEmail: string) {
  await page.route('**/api/v1/users/**', async (route) => {
    if (route.request().method() !== 'PUT') {
      await route.continue()
      return
    }
    await fulfillJson(route, 200, {
      success: true,
      data: { id: 'user-1', email: updatedEmail },
    })
  })
}
```

The method guard (`route.request().method() !== 'PUT'`) passes through any non-PUT requests to the same URL pattern unchanged, preventing unintended interception.

All selectors use semantic Playwright queries. The profile button matches the pattern `/profile|プロフィール|edit profile/i` so the test is not brittle to the exact label wording — a practical choice while the label text might still be refined.

---

## Test Coverage Summary

| Layer | File | Tier | Count |
|---|---|---|---|
| Backend validation | `user-profile-validation.small.test.ts` | Small | 19 |
| Backend HTTP handler | `user-profile.medium.test.ts` | Medium | 16 |
| Frontend navigation | `navigation.small.test.ts` | Small | 4 |
| Frontend component | `UserProfilePage.medium.test.tsx` | Medium | 12 |
| Browser flow | `user-profile.medium.test.ts` (Playwright) | E2E | 5 |
| **Total** | | | **56** |

All 56 tests were written before any production code (Red phase), and all 56 turned green after `e0ae283` (Green phase).

---

## Known Limitation

The current `PUT /api/v1/users/:userId` implementation **echoes the submitted email back without persisting it to a database**. The handler takes `userId` from the path parameter and returns it alongside the new email, but does not update any stored user record. Real persistence — updating the user's email and hashing the new password in the DB — is a follow-up task. The API contract and the full test suite are in place, so adding persistence means extending the handler without needing to touch the tests.

---

## Summary

| Aspect | Detail |
|---|---|
| New production files | `user-profile-validation.ts`, `UserProfilePage.tsx` |
| Modified production files | `hono-app.ts`, `navigation.ts`, `App.tsx` |
| New test files | 5 (validation small, backend medium, navigation small, component medium, Playwright E2E) |
| Total tests | 56 (all written before any production code) |
| TDD phases | Red (56 failing) → Green (minimal implementation) |
| State management | `useAtom(authAtom)` reads current email, writes updated email on 200 response; `atomWithStorage` persists to `localStorage` |
| Password change UX | "現在のパスワード" field renders only when "New Password" is non-empty |
| Known limitation | No DB persistence; echoes submitted values; tracked as follow-up |

The discipline of writing 56 tests before a single line of production code pays off in two ways: the implementation is shaped entirely by the observable behaviour defined in the tests, and the resulting component tree has explicit contracts at every boundary — validation logic, HTTP handler, Jotai atom updates, and the full browser flow — all independently verifiable.
