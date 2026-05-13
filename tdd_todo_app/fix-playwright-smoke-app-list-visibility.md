# Fixing Playwright Smoke Failure: AppList Visibility

## Context

The Playwright smoke test in `frontend/e2e/crud.spec.ts` intermittently failed at:

```ts
await expect(page.getByText('No apps yet. Create your first app!')).toBeVisible();
```

After auth screens were introduced, the app could be authenticated while page state remained `landing`, `login`, or `signup`.

## Root cause

For authenticated users, `App.tsx` mounts `AppListPage`, but `AppListPage` previously rendered only when:

- `currentPage.name === 'app-list'`

It also enabled the apps query only in that state.  
So when auth existed but page state was still `landing/login/signup`, `AppListPage` returned `null`, and the empty-state text was never rendered.

## Fix implemented

File changed: `frontend/src/features/app-list/pages/AppListPage.tsx`

1. Introduced `shouldShowAppList`
2. Enabled data fetching when page is one of:
   - `app-list`
   - `landing`
   - `login`
   - `signup`
3. Rendered AppList UI for those states instead of returning `null`

Key change:

```ts
const shouldShowAppList =
  currentPage.name === 'app-list' ||
  currentPage.name === 'landing' ||
  currentPage.name === 'login' ||
  currentPage.name === 'signup'

const { data, isLoading, isError } = useGetApiV1Apps({
  query: { enabled: shouldShowAppList },
})

if (!shouldShowAppList) return null
```

## Result

Smoke tests are green again:

```bash
npx playwright test --grep "@smoke" --project=chromium
```

Result: **2 passed** (`example.spec.ts`, `crud.spec.ts`).
