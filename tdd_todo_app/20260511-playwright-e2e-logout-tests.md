# Playwright E2E Tests for the Logout Feature

**Commit:** `0b72636` — `test(e2e): add Playwright logout E2E tests`  
**File:** `frontend/e2e/logout.medium.test.ts`

---

## Target Readers

Frontend/fullstack engineers who are writing or expanding Playwright E2E test suites and want to understand practical patterns for testing authentication flows — specifically how to arrange a pre-authenticated state without hitting a real backend.

---

## What This Article Covers

- Why E2E tests for logout were needed
- How `localStorage` is seeded before navigation to simulate an authenticated user
- How `page.route()` is used to stub API responses
- The two test scenarios and their structure
- Helper-function patterns that keep the test file readable

This article does **not** cover the implementation of the logout feature itself (that was covered in an earlier commit), Playwright project configuration, or CI integration.

---

## Background

The Todo App TDD project uses [jotai](https://jotai.org/) `atomWithStorage` to persist auth state in `localStorage` under the key `'auth'`. When the app boots with that key present, the user lands directly on the authenticated view (app-list screen). When the user clicks "ログアウト", the atom is cleared and the router redirects to the LoginPage.

The logout feature already had unit and integration coverage, but there was no E2E test that exercised the full browser flow — clicking the button, asserting the redirect, and verifying that re-login restores the session. That gap was the motivation for this work.

---

## Core Problem: How to Start a Test as an Authenticated User

The naive approach is to drive through the login form in every test that needs an authenticated start. The downside is obvious: it couples every test to the login flow, slows down the suite, and breaks when the login API has unrelated issues.

Playwright's [`page.addInitScript()`](https://playwright.dev/docs/api/class-page#page-add-init-script) solves this elegantly. Any script registered with `addInitScript` runs inside the page context **before** the first navigation, which means `localStorage` can be written before the React app mounts and reads from it.

```typescript
async function seedAuthState(page: Page) {
  await page.addInitScript(() => {
    const authState = {
      token: 'test-token',
      user: { id: 'user-1', email: 'user@example.com' },
    };
    localStorage.setItem('auth', JSON.stringify(authState));
  });
}
```

The key `'auth'` matches exactly what jotai's `atomWithStorage` reads on startup. After calling `await page.goto('/')`, the app finds the token and user in storage and renders the authenticated view — no login form interaction required.

---

## Stubbing API Responses with `page.route()`

Two API endpoints are called during the authenticated view:

| Endpoint | Method | Purpose |
|---|---|---|
| `GET /api/v1/apps` | GET | Load the user's app list |
| `POST /api/v1/auth/login` | POST | Submit the login form (re-login scenario only) |

Both are stubbed with `page.route()` so the tests run without a real backend:

```typescript
async function registerAppsListStub(page: Page) {
  await page.route('**/api/v1/apps', async (route) => {
    if (route.request().method() !== 'GET') {
      await route.continue();
      return;
    }
    await fulfillJson(route, 200, { success: true, data: [] });
  });
}

async function registerLoginStub(page: Page) {
  await page.route('**/api/v1/auth/login', async (route) => {
    if (route.request().method() !== 'POST') {
      await route.continue();
      return;
    }
    await fulfillJson(route, 200, {
      success: true,
      data: {
        token: 'test-token',
        user: { id: 'user-1', email: 'user@example.com' },
      },
    });
  });
}
```

A shared `fulfillJson` helper keeps the boilerplate (`status`, `contentType`, `JSON.stringify`) in one place:

```typescript
async function fulfillJson(route: Route, status: number, body: unknown) {
  await route.fulfill({
    status,
    contentType: 'application/json',
    body: JSON.stringify(body),
  });
}
```

Note the guard on HTTP method (`route.request().method()`). Without it, all requests matching the URL pattern — including preflight requests — would be intercepted and incorrectly fulfilled. Calling `route.continue()` for unmatched methods lets those requests pass through unchanged.

---

## The Two Test Scenarios

### 1. Logout Happy Path

Verifies that clicking "ログアウト" redirects to the login page and removes the logout button.

```typescript
test(
  'logout happy path: when the logout button is clicked, then the user is redirected to the login page and the logout button disappears',
  async ({ page }) => {
    // Arrange — start as an authenticated user
    await seedAuthState(page);
    await registerAppsListStub(page);
    await page.goto('/');
    await expectAuthenticatedView(page);

    // Act — click the logout button
    await page.getByRole('button', { name: 'ログアウト' }).click();

    // Assert — login page is shown, logout button is gone
    await expect(page.getByRole('heading', { name: 'Login' })).toBeVisible();
    await expect(page.getByRole('button', { name: 'ログアウト' })).toBeHidden();
  },
);
```

The AAA (Arrange / Act / Assert) structure is made explicit with comments. `expectAuthenticatedView` is a shared assertion helper (see below).

### 2. Re-Login after Logout

Verifies the full round-trip: log out → fill the login form → assert the authenticated view is restored.

```typescript
test(
  'logout then re-login: after logging out, the user can log in again and return to the authenticated view',
  async ({ page }) => {
    // Arrange — start as an authenticated user, then log out
    await seedAuthState(page);
    await registerAppsListStub(page);
    await registerLoginStub(page);
    await page.goto('/');
    await expectAuthenticatedView(page);

    await page.getByRole('button', { name: 'ログアウト' }).click();
    await expect(page.getByRole('heading', { name: 'Login' })).toBeVisible();

    // Act — log in again
    await page.getByLabel('Email').fill('user@example.com');
    await page.getByLabel('Password').fill('password123');
    await page.getByRole('button', { name: 'Login' }).click();

    // Assert — back to the authenticated app-list view
    await expectAuthenticatedView(page);
    await expect(page.getByText('No apps yet. Create your first app!')).toBeVisible();
  },
);
```

This test is deliberately more involved. Logging out and re-logging in in the same browser session means the `localStorage` key is cleared and then written again by the real app code — the stub covers the POST request that the login form makes, not the state management itself.

---

## Helper Functions

All helpers live in the same file as the tests. Keeping them co-located avoids the overhead of shared fixture files for a single test module while still making the test bodies readable.

| Helper | Role |
|---|---|
| `fulfillJson` | Low-level route fulfillment with JSON content type |
| `registerAppsListStub` | Stubs `GET /api/v1/apps` → empty list |
| `registerLoginStub` | Stubs `POST /api/v1/auth/login` → success response |
| `seedAuthState` | Seeds `localStorage` with auth state before navigation |
| `expectAuthenticatedView` | Asserts heading "Todo App TDD" and "ログアウト" button are visible |

---

## Selector Strategy: No `data-testid`

All selectors use Playwright's semantic query methods:

- `getByRole('heading', { name: 'Todo App TDD' })` — matches the `<h1>` element
- `getByRole('button', { name: 'ログアウト' })` — matches the button by its accessible label
- `getByLabel('Email')` / `getByLabel('Password')` — matches form inputs by their `<label>` text
- `getByText('No apps yet...')` — matches visible text content

This approach avoids scattering `data-testid` attributes across production components and keeps tests coupled to the UI's visible, accessible structure rather than internal implementation details.

---

## Test Results

```
2 passed (7.6s)  [chromium]
```

Both scenarios pass against the Chromium browser in the Playwright project. Runtime is fast because there are no real network requests — every backend call is intercepted before it leaves the browser.

---

## Summary

| Concern | Approach |
|---|---|
| Arrange authenticated state | `page.addInitScript()` seeds `localStorage['auth']` before navigation |
| Isolate from real backend | `page.route()` stubs `GET /api/v1/apps` and `POST /api/v1/auth/login` |
| Test structure | AAA with inline comments |
| Selectors | Role / label / text — no `data-testid` |
| Helper placement | Co-located in the same test file |

The key insight is the combination of `addInitScript` + `page.route()`: the former replaces the need to drive through the login form, and the latter replaces the need for a live backend. Together they make each test fast, deterministic, and focused on the behavior under test — the logout flow itself.
