# Aligning AppListPage Unit Tests with Auth-Aware Routing

## Background

We recently fixed a Playwright smoke failure by updating `AppListPage` to stay visible for auth bootstrap states, not only `app-list`.  
`frontend/src/features/app-list/pages/AppListPage.tsx` now uses `shouldShowAppList` for:

- `app-list`
- `landing`
- `login`
- `signup`

This stabilized smoke behavior where auth can already exist while page state is still transitioning.

## Problem

After that change, one unit test still asserted the old rule:

- “if currentPage is not `app-list`, render nothing”

That expectation was no longer correct for `landing/login/signup`, so CI failed in:

- `frontend/src/features/app-list/pages/AppListPage.test.tsx`

## Fix

Updated the test suite to match intended behavior:

1. Replaced outdated case with:
   - when currentPage is `landing`, `AppListPage` still renders
2. Added an explicit non-render case where null is still expected:
   - when currentPage is `app-detail`, component renders nothing

This keeps the test contract precise: auth-entry states render the list, detail/edit/create subpages do not.

## Outcome

`npm test` is green again:

- **16 files passed**
- **172 tests passed**

## Why this matters

The change prevents false negatives in CI and keeps unit tests aligned with the actual routing model.  
By encoding both “should render” and “should not render” states explicitly, future routing changes are less likely to reintroduce regressions silently.
