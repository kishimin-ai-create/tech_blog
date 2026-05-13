# Adding Authentication Screens with TDD: Red → Green → Refactor → Review

**Target readers:** Frontend developers who want to see TDD applied to real UI features — auth forms, shared hooks, and state management with Jotai.

---

## Overview

The TDD Todo App previously had no authentication gate — any visitor landed straight in the app. This post walks through the full TDD cycle used to add three auth screens (Landing, Login, Signup), wire up persistent auth state, and fix five real bugs discovered during a structured code review — all with 171 tests passing at the end.

Tech stack: React, TypeScript, Jotai, Tailwind CSS, Vitest, React Testing Library, MSW.

---

## Red Phase: Tests First

Before writing a single line of production code, tests were written for the three new pages and for `App.tsx` routing behaviour. They all failed, which was the goal.

Key things the tests specified upfront:

- `LandingPage` renders a **Login** button and a **Sign Up** button
- `LoginPage` posts credentials to `POST /api/v1/auth/login` and, on success, populates `authAtom` and navigates to `app-list`
- `SignupPage` does the same against `POST /api/v1/auth/signup`
- `App.tsx` shows auth screens when `authAtom` is `null`, and app screens when authenticated

MSW handlers mocked the API responses so the tests stayed fast and isolated from the real backend.

---

## Green Phase: Minimal Implementation

With the failing tests as the spec, the minimal production code was written:

**Auth state** — a single `atomWithStorage` from Jotai persists the token and user to `localStorage`:

```ts
// frontend/src/shared/auth.ts
import { atomWithStorage } from 'jotai/utils'

export type AuthState = {
  token: string
  user: { id: string; email: string }
}

export const authAtom = atomWithStorage<AuthState | null>('auth', null)
```

Using `atomWithStorage` was a deliberate choice: E2E tests can seed auth state by calling `localStorage.setItem('auth', JSON.stringify({ token, user }))` before the app loads, without needing a real login flow.

**Auth gating in App.tsx** — pure conditional rendering, no `useEffect`:

```tsx
function App() {
  const [auth] = useAtom(authAtom)
  const [currentPage] = useAtom(currentPageAtom)

  if (!auth) {
    if (currentPage.name === 'signup') return <SignupPage />
    if (currentPage.name === 'login') return <LoginPage />
    return <LandingPage />
  }

  return (
    <div>
      <AppListPage />
      {currentPage.name === 'app-detail' && <AppDetailPage appId={currentPage.appId} />}
      {currentPage.name === 'app-create' && <AppCreatePage />}
      {currentPage.name === 'app-edit' && <AppEditPage appId={currentPage.appId} />}
    </div>
  )
}
```

`fetch` was used directly (no axios or orval) because the auth endpoints weren't in the OpenAPI spec, keeping the implementation simple and dependency-free.

---

## Refactor Phase: Extract `useAuthForm`

`LoginPage` and `SignupPage` had nearly identical form logic — the same `email` state, `password` state, `error` state, and `fetch` call. The Refactor phase eliminated this duplication by extracting a shared `useAuthForm` hook:

```ts
// frontend/src/features/auth/hooks/useAuthForm.ts
export function useAuthForm({ endpoint, fallbackErrorMessage, onSuccess }: UseAuthFormOptions) {
  const [email, setEmail] = useState('')
  const [password, setPassword] = useState('')
  const [error, setError] = useState<string | null>(null)
  const [isSubmitting, setIsSubmitting] = useState(false)

  async function submitAuthRequest() {
    if (isSubmitting) return                          // re-entry guard
    if (!email.trim() || !password) {
      setError('Email and password are required')
      return
    }
    setIsSubmitting(true)
    try {
      const response = await fetch(endpoint, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ email, password }),
      })
      const data = (await response.json()) as AuthResponse
      if (!response.ok || !data.success) {
        setError(data.success ? fallbackErrorMessage : data.error)
        return
      }
      onSuccess(data.data)
    } catch {
      setError('An unexpected error occurred')
    } finally {
      setIsSubmitting(false)
    }
  }

  const handleSubmit = (e: React.FormEvent) => { e.preventDefault(); void submitAuthRequest() }
  return { email, setEmail, password, setPassword, error, isSubmitting, handleSubmit }
}
```

Each page then becomes a thin consumer — just configure the endpoint and define `onSuccess`:

```tsx
const { email, setEmail, password, setPassword, error, isSubmitting, handleSubmit } = useAuthForm({
  endpoint: '/api/v1/auth/login',
  fallbackErrorMessage: 'Login failed',
  onSuccess: (auth) => { setAuth(auth); goToAppList() },
})
```

---

## Review Phase: Five Bugs Found and Fixed

A structured code review ran after the tests were green. Five issues were surfaced:

| Priority | Issue | Fix |
|---|---|---|
| P1 | **Double-submit** — no guard while fetch was in flight | Added `isSubmitting` state; button disabled and relabelled ("Logging in…") during submission |
| P1 | **Navigation buttons missing `type="button"`** — HTML default is `type="submit"` | Added explicit `type="button"` to all nav buttons outside `<form>` |
| P2 | **`currentPageAtom` initial value was `'app-list'`** — atom and rendered page were out of sync on first load | Changed initial value to `{ name: 'landing' }` to make state honest |
| P2 | **No client-side validation** — empty credentials reached the server | Added trim/empty guard in `submitAuthRequest` before the `fetch` call |
| P3 | **Network error branch untested** — the `catch` path had zero coverage | Added MSW `HttpResponse.error()` tests to both `LoginPage.test.tsx` and `SignupPage.test.tsx` |

The P1 bugs in particular would have been invisible in happy-path manual testing — they only surface under specific timing or browser conditions. The Refactor phase had also silently introduced the `isSubmitting` fix by centralising the logic, making it easy to add the guard in one place rather than two.

The `localStorage` / XSS risk (P2) was acknowledged but deferred: the spec explicitly requires `atomWithStorage` for E2E seed compatibility, so a migration to `httpOnly` cookies needs backend coordination and is tracked as a known risk.

---

## Final State

- **3 new pages**: `LandingPage`, `LoginPage`, `SignupPage`
- **1 shared hook**: `useAuthForm` — used by both form pages, zero logic duplication
- **Auth state**: persisted to `localStorage` via `atomWithStorage`, readable by E2E tests
- **Routing**: pure conditional rendering in `App.tsx`, no effect-based redirects
- **Tests**: 171 passing across 16 test files

---

## Takeaways

1. **Write tests before pages.** It forces you to define the full API contract (what state gets set, what navigation happens) before implementation details distract you.
2. **The Refactor phase is where design happens.** Extracting `useAuthForm` after two pages existed made the right abstraction obvious — not a premature guess.
3. **A review phase catches what tests miss.** The double-submit bug and the `type="button"` omission were both silent under normal testing but real risks under adversarial conditions.
4. **State consistency matters even when behaviour looks correct.** `currentPageAtom` initialising to `'app-list'` never caused a visible bug, but it was a lying atom — and lying state is technical debt waiting to become a real bug.
