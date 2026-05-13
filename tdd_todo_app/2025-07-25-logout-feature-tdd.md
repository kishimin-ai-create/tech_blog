# Implementing a Logout Feature with TDD in React + Jotai

**Date:** 2025-07-25  
**Target readers:** Frontend engineers familiar with React and TypeScript; developers learning TDD workflows  
**Scope:** This article covers the full Red → Green → Refactor → Code Review cycle for a logout feature backed by Jotai state management. It does not cover server-side token revocation (acknowledged as a follow-up task).

---

## Background

The TDD todo app already had signup and login flows built around Jotai atoms. When a user authenticates, their token and user profile are stored in `authAtom` (an `atomWithStorage` atom) and the active page is tracked in `currentPageAtom`. What was missing was a way for the user to get _out_: no logout hook, no logout button, nothing in the authenticated layout to trigger session teardown.

The goal was straightforward — clear the auth state, redirect to the login page, and persist that cleared state across page reloads — but the path to get there was deliberately methodical: tests first, implementation second.

---

## The TDD Cycle

### Red Phase — Write Failing Tests

Before writing a single line of production code, three test files were created:

| File | Size tier | Purpose |
|---|---|---|
| `useLogout.small.test.ts` | Small (unit) | Hook API surface, atom state transitions, boundary cases |
| `LogoutButton.small.test.tsx` | Small (unit) | Rendering and click interaction, with `useLogout` mocked |
| `LogoutButton.medium.test.tsx` | Medium (integration) | Full `App` render with a real Jotai store |

At this point `useLogout.ts` and `LogoutButton.tsx` did not exist. Every test failed with module resolution errors. That was expected — the failing tests define the contract.

The unit tests for the hook covered four scenarios up front:

```ts
// Happy path
it('when logout is called with an existing auth state, then authAtom is set to null', ...)
it('when logout is called with an existing auth state, then currentPageAtom is set to { name: "login" }', ...)

// Navigation from any page
it('when logout is called from the app-detail page, then currentPageAtom becomes { name: "login" }', ...)
it('when logout is called from the app-create page, then currentPageAtom becomes { name: "login" }', ...)

// Boundary case
it('when logout is called and authAtom is already null, then authAtom remains null', ...)
it('when logout is called and authAtom is already null, then currentPageAtom still becomes { name: "login" }', ...)
```

The integration tests verified observable UI behaviour from the outside:

```ts
it('when the logout button is clicked, then authAtom is set to null', ...)
it('when the logout button is clicked, then currentPageAtom becomes { name: "login" }', ...)
it('when the logout button is clicked, then App renders the LoginPage with an email input', ...)
it('when the logout button is clicked, then the "ログアウト" button is no longer in the document', ...)
```

Seventeen tests total. All red.

---

### Green Phase — Implement the Minimum

With the contracts defined by the tests, the implementation turned out to be compact.

**`useLogout.ts`**

```ts
import { useSetAtom } from 'jotai'

import { authAtom } from '../../../shared/auth'
import { currentPageAtom } from '../../../shared/navigation'

/**
 * Hook that provides a logout function.
 * Clears the auth state and navigates to the login page.
 */
export function useLogout() {
  const setAuth = useSetAtom(authAtom)
  const setCurrentPage = useSetAtom(currentPageAtom)

  function logout() {
    setAuth(null)
    setCurrentPage({ name: 'login' })
  }

  return { logout }
}
```

Two things worth noting here:

1. **`useSetAtom` instead of `useAtom`** — The hook only needs to write; it never reads the current auth value. `useSetAtom` avoids subscribing the component to state changes it doesn't care about and makes the intent explicit.

2. **`atomWithStorage` and `null`** — `authAtom` is defined as `atomWithStorage<AuthState | null>('auth', null)`. When `setAuth(null)` is called, Jotai's `atomWithStorage` implementation writes `null` to `localStorage` under the `'auth'` key. On the next page load, the atom initialises to `null` (the stored value), so the session is properly terminated even after a browser refresh.

**`LogoutButton.tsx`**

```tsx
import { useLogout } from '../hooks/useLogout'

/**
 * Button component that logs the current user out.
 * Calls the logout function from useLogout on click.
 */
export function LogoutButton() {
  const { logout } = useLogout()

  return (
    <button
      type="button"
      onClick={logout}
      className="rounded bg-gray-200 px-4 py-2 text-sm font-medium hover:bg-gray-300 focus-visible:outline focus-visible:outline-2 focus-visible:outline-gray-400 transition-colors duration-150"
    >
      ログアウト
    </button>
  )
}
```

**`App.tsx` — Adding the button to the authenticated layout**

```tsx
return (
  <div>
    <LogoutButton />
    <AppListPage />
    {currentPage.name === 'app-detail' && <AppDetailPage appId={currentPage.appId} />}
    {currentPage.name === 'app-create' && <AppCreatePage />}
    {currentPage.name === 'app-edit' && <AppEditPage appId={currentPage.appId} />}
  </div>
)
```

`LogoutButton` is rendered unconditionally inside the authenticated branch of `App`. It is never rendered when `auth` is `null`, so there is no risk of showing it to unauthenticated users.

All 17 tests now passed. Green.

---

### Refactor Phase — Quality Improvements

With tests green, the focus shifted to code quality without changing behaviour:

- Added JSDoc comments to both `useLogout` and `LogoutButton` to satisfy the project's `jsdoc/require-jsdoc` ESLint rule (`publicOnly: true`, `FunctionDeclaration: true`).
- Applied Tailwind classes to `LogoutButton` for consistent styling with the rest of the UI.
- Added `type="button"` to the `<button>` element to prevent accidental form submission if the button is ever placed inside a `<form>`.

The tests stayed green throughout.

---

## Test Structure Deep Dive

### Small Tests — Isolating Units

The unit tests for `LogoutButton` mock `useLogout` entirely:

```ts
vi.mock('../hooks/useLogout', () => ({
  useLogout: vi.fn(),
}))

// eslint-disable-next-line import/order -- placed here for readability; vi.mock is hoisted by Vitest regardless
import { useLogout } from '../hooks/useLogout'
```

The `vi.mock()` call is hoisted to the top of the module by Vitest's compiler transform, so the position of the `import` statement in source order doesn't affect whether the mock is active. The `eslint-disable-next-line` comment documents this fact directly at the relevant line.

The unit tests for `useLogout` use a real Jotai store isolated per test:

```ts
beforeEach(() => {
  store = createStore()
  localStorage.clear()  // atomWithStorage syncs to localStorage; prevent leakage
})

function createWrapper(store: TestStore) {
  return function Wrapper({ children }: { children: ReactNode }) {
    return createElement(Provider, { store }, children)
  }
}
```

Each test gets its own `createStore()` instance. `localStorage.clear()` runs before each test because `atomWithStorage` reads from and writes to `localStorage` as a side effect — without clearing, atoms from a previous test could bleed into the next one.

### Medium Tests — Integration via Real Store

The integration tests render the full `App` component with a pre-seeded store, making assertions at the UI level rather than the atom level (though both are verified):

```ts
it('when the logout button is clicked, then App renders the LoginPage with an email input', async () => {
  const user = userEvent.setup()
  const store = createStore()
  store.set(authAtom, AUTHENTICATED_USER)
  renderWithProviders(<App />, { store, initialPage: { name: 'app-list' } })

  await user.click(await screen.findByRole('button', { name: 'ログアウト' }))

  await waitFor(() => {
    expect(screen.getByRole('textbox', { name: /email/i })).toBeInTheDocument()
  })
})
```

The MSW server stubs the `GET /api/v1/apps` endpoint so `AppListPage` renders without a real backend. This keeps integration tests hermetic while still exercising the real component tree.

---

## Code Review Findings

After implementation and refactor, a structured code review surfaced four findings. Three were fixed immediately; one was risk-accepted as a tracked follow-up.

### Fixed: Arrow wrapper removed from `onClick`

The initial implementation used:

```tsx
onClick={() => { logout() }}
```

This creates a new function reference on every render for no benefit, since `logout` is a `() => void` function taking no arguments. The fix is direct assignment:

```tsx
onClick={logout}
```

There is also a subtle future footgun here: if `logout` were ever made async, the arrow wrapper would silently discard the returned `Promise`. Passing the reference directly makes the intent clear.

Alongside this fix, the corresponding test assertion was updated from `.toHaveBeenCalledWith()` (zero-args check) to `.toHaveBeenCalled()`. With direct assignment, React passes the `SyntheticEvent` to the handler, so checking for zero arguments would fail — the presence check captures the actual intent.

### Fixed: `import/order` rule scoped correctly

An earlier version of `eslint.config.js` disabled `import/order` globally across all test files to accommodate the `vi.mock` / `import` ordering pattern. The review identified this as overly broad: the rule was silenced for the entire growing test suite when the actual exception was a single import line in a single file.

The fix replaced the global override with a targeted inline disable on the exact line where the rule fires:

```ts
// eslint-disable-next-line import/order -- placed here for readability; vi.mock is hoisted by Vitest regardless
import { useLogout } from '../hooks/useLogout'
```

Import ordering discipline is now enforced across all other test files.

### Fixed: Redundant "accessible role" test removed

The test suite originally included two describe blocks in `LogoutButton.small.test.tsx`:

```ts
// Test A
expect(screen.getByRole('button', { name: 'ログアウト' })).toBeInTheDocument()

// Test B (redundant)
expect(screen.getByRole('button')).toBeInTheDocument()
```

Test B is a strict subset of Test A. If the button renders correctly with the right label, the role check is already implied. If the element is rendered as a `<div>` instead of `<button>`, both tests fail simultaneously. Test B provides zero independent signal and was removed.

### Risk Accepted: No server-side token revocation

`useLogout` only clears client-side state. No `POST /api/v1/auth/logout` endpoint exists on the backend, and none was added. The JWT stored in `localStorage['auth']` continues to be accepted by the server until its natural expiry — clicking "ログアウト" does not invalidate it on the backend.

If a token were exfiltrated before logout (via XSS or network interception), the client-side logout would provide no protection. The acknowledged mitigations are:

1. Implement `POST /api/v1/auth/logout` with server-side token blacklisting — tracked as a follow-up task before production.
2. Keep token expiry short (≤ 15 min) to bound the attack window in the interim.
3. If stateless JWTs are required, issue refresh tokens with a server-managed revocation list.

This is an architecture-level constraint, not something a single feature branch can fully resolve. The risk is documented and accepted with a short expiry as mitigation.

---

## Summary

| Aspect | Detail |
|---|---|
| New files | `useLogout.ts`, `LogoutButton.tsx`, 3 test files |
| Modified files | `App.tsx`, `eslint.config.js` |
| Test count | 17 tests (7 hook unit, 5 component unit, 5 integration) |
| State management | Jotai `useSetAtom`; `atomWithStorage` null-clears localStorage |
| TDD phases | Red (17 failing tests) → Green (minimal implementation) → Refactor (docs, styles, type safety) → Code Review (4 findings, 3 fixed) |
| Known limitation | No server-side token revocation; mitigated by short JWT expiry |

The implementation is minimal by design: two production files, a clean hook-component separation, and a test suite that exercises both units in isolation and together through the real component tree. The code review pass caught real problems — a lint scope bug that would have silently spread, a redundant test case, and a subtle render-performance issue — making the final code meaningfully better than the first-pass green state.
