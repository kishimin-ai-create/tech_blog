# Devlog 2026-05-10: The Bug That Wasn't There — Chrome Autofill, a Repo Migration, and a Render Deployment Mystery

**Date**: 2026-05-10  
**Stack**: React + TypeScript + Vite + Jotai + Hono + Vitest + Render  
**Commit**: `bc16cad` — `fix: add autoComplete attributes to auth forms to prevent browser autofill`

---

## Overview

Three separate things got resolved in today's session:

1. **A bug report about the Sign Up form retaining values across navigation** — which turned out to be caused by Chrome's autofill, not application state
2. **A GitHub repository migration** — moving from a personal account to an organization account, with consequences for CI and deployment
3. **A Render deployment confusion** — where `yarn start` failures in the logs were misleading because the backend was actually starting correctly

---

## 1. The Bug That Wasn't There (Until Chrome Made It Real)

### The Report

> "When I type values into the Sign Up form, navigate to Login, then go back to Sign Up — my previous values are still in the fields."

Classic form state leak. Likely a `useState` living in global scope, or maybe a missing `key` prop on a conditionally rendered component. Time to investigate.

### Investigation: All Signs Pointed to "No Bug"

The form state lives entirely in `useAuthForm.ts`:

```ts
// frontend/src/features/auth/hooks/useAuthForm.ts
export function useAuthForm({ endpoint, fallbackErrorMessage, onSuccess }) {
  const [email, setEmail]       = useState('')
  const [password, setPassword] = useState('')
  // ...
}
```

Local `useState` — no Jotai atom, no global scope. This state dies when the component unmounts.

Next, `App.tsx` was checked:

```tsx
// frontend/src/App.tsx
if (!auth) {
  if (currentPage.name === 'signup') return <SignupPage key="signup" />
  if (currentPage.name === 'login')  return <LoginPage  key="login"  />
  return <LandingPage />
}
```

Two things that should guarantee a clean mount on every navigation:

- `SignupPage` and `LoginPage` are **different component types** — React unmounts the old one and mounts a fresh instance whenever the type changes
- Both have explicit `key` props (`key="signup"`, `key="login"`) — a defensive measure added in a previous commit (`ca13d16`) specifically to ensure state independence even if the two pages were ever refactored into a single shared component

Running all 180 tests confirmed everything passes, including the regression test in `App.test.tsx` that exercises the exact `signup → login → signup` navigation flow and asserts the fields are empty at each step.

**Conclusion from code review: the application state is working correctly.**

### The Real Root Cause: Chrome's Credential Manager

The actual culprit was **browser autofill firing on React remount**.

Here's what happens in Chrome when navigating between pages in this SPA:

1. User types an email and password on the Sign Up page
2. Chrome's credential manager registers the new `<input type="email">` and `<input type="password">` elements
3. User navigates to Login — React unmounts `SignupPage`, mounts `LoginPage`
4. User navigates back to Sign Up — React unmounts `LoginPage`, mounts `SignupPage`
5. **Chrome sees new `<input>` elements appear at the same URL without a page reload**
6. Chrome triggers `onChange` events on the controlled inputs, injecting the previously typed (or previously saved) values
7. React processes these synthetic events and updates `useState` accordingly

Because the inputs are controlled (`value={email}` + `onChange`), Chrome's autofill works by dispatching `onChange` — not by directly mutating the DOM. This makes it indistinguishable from actual user input from React's perspective. The `useState` values were genuinely zero on mount; it was autofill writing into them immediately after mount.

This is a known SPA-specific behavior: a traditional page navigation flushes browser state, but a React remount at the same URL does not.

### The Fix: `autoComplete` Attributes

The fix is small — just two attributes per form — but the choice of values matters.

**`SignupPage.tsx`**

```tsx
<input
  id="email"
  type="email"
  value={email}
  onChange={(e) => setEmail(e.target.value)}
  autoComplete="off"          // ← added
/>
<input
  id="password"
  type="password"
  value={password}
  onChange={(e) => setPassword(e.target.value)}
  autoComplete="new-password" // ← added
/>
```

**`LoginPage.tsx`**

```tsx
<input
  id="email"
  type="email"
  value={email}
  onChange={(e) => setEmail(e.target.value)}
  autoComplete="username"         // ← added
/>
<input
  id="password"
  type="password"
  value={password}
  onChange={(e) => setPassword(e.target.value)}
  autoComplete="current-password" // ← added
/>
```

### Why These Specific Values?

| Field | Value | Reason |
|---|---|---|
| SignupPage email | `"off"` | Suppress autofill on a new account form — no saved credential should prefill here |
| SignupPage password | `"new-password"` | Signals to Chrome that this is an account creation form. Chrome will offer to save the new credential but **will not inject a previously saved one** |
| LoginPage email | `"username"` | Correct semantic value for login email. Tells the browser's password manager to match saved credentials |
| LoginPage password | `"current-password"` | Works in conjunction with `username` to let the browser's password manager autofill on the login page — which is the intended UX |

The key insight: `new-password` vs. `current-password` is the semantic signal Chrome uses to distinguish "create account" from "sign in". Setting it correctly solves the autofill-on-signup problem while preserving the intended autofill behavior on login.

### Result

- 28/28 auth-related tests pass after the fix
- All 180 tests pass overall
- No test changes were needed: the test environment (Vitest + jsdom) does not emulate browser autofill, so the existing regression tests already covered the underlying React state behavior correctly

### Lesson

When all 180 unit/integration tests pass and the React code is correct, but the bug is still visible in the real browser — check the browser. In a SPA that remounts components without a full page reload, Chrome's credential manager can inject values into controlled inputs via `onChange`. The fix is proper `autoComplete` semantics, not application state changes.

---

## 2. GitHub Repository Migration

The repository was transferred from the personal account `Kazuma-Ishimine/TDD_todo_app` to the organization `kishimin-ai-create/TDD_todo_app`. The local remote was updated to match:

```bash
git remote set-url origin git@github.com:kishimin-ai-create/TDD_todo_app.git
```

**Impact on CI (GitHub Actions)**: None. The workflow files live inside the repository itself and move with it. Actions continue to run without reconfiguration.

**Impact on Render deployment**: Potentially significant. Render's GitHub App is authorized at the organization level. When a repository moves from a personal account to an org (or vice versa), Render may lose webhook access to trigger auto-deploys. The GitHub App needs to be re-authorized for the new org in GitHub's settings. If auto-deploys stop firing after the migration, this is the first place to check.

---

## 3. Render Deployment Troubleshooting

Render's deploy logs showed a `yarn start` failure before a successful startup. This was confusing because `render.yaml` defines the start command explicitly:

```yaml
# render.yaml
services:
  - type: web
    name: tdd-todo-app-backend
    env: node
    buildCommand: cd backend && npm ci --include=dev && npm run build && node dist/infrastructure/migrate.mjs
    startCommand: cd backend && npm start   # ← what should be used
```

The `yarn start` failure came from **an old or manually configured service in the Render dashboard**, where the Start Command field had not been updated to match `render.yaml`. Render can have a disconnect between the `render.yaml` specification and what is saved in the service's dashboard settings if the service was originally created manually.

**The backend was already running successfully at port 10000** — the `yarn start` failure was from a pre-flight check or an old config being tried first, not the actual service start.

The fix: navigate to the Render dashboard → the backend service → Settings → Start Command, and ensure it reads `cd backend && npm start`, matching `render.yaml`. After alignment, the error disappears from the logs.

---

## Summary

| Topic | Root Cause | Fix |
|---|---|---|
| Sign Up form autofill | Chrome's credential manager dispatches `onChange` on controlled inputs when React remounts them in a SPA | Add correct `autoComplete` attributes: `off`/`new-password` on signup, `username`/`current-password` on login |
| Repo migration | Personal → org GitHub transfer | Update local remote URL; re-authorize Render's GitHub App for the new org |
| Render `yarn start` error | Dashboard Start Command out of sync with `render.yaml` | Align dashboard setting to `cd backend && npm start` |

The most reusable takeaway from today: **browser behavior and application state are separate layers**. When the code and tests are correct but a bug persists in the browser, the browser itself is the next debugging target — and `autoComplete` is one of the levers that controls how aggressively it interacts with your forms.
