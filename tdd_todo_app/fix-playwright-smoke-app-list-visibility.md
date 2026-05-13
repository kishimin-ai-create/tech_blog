# Playwright Smoke テスト失敗の修正：AppList の表示

## 背景

`frontend/e2e/crud.spec.ts` の Playwright smoke テストが断続的に以下で失敗していました：

```ts
await expect(page.getByText('No apps yet. Create your first app!')).toBeVisible();
```

認証画面が導入された後、アプリが認証済みであってもページの状態が `landing`、`login`、または `signup` のままになることがありました。

## 根本原因

認証済みユーザーの場合、`App.tsx` は `AppListPage` をマウントしますが、`AppListPage` は以前は以下の場合のみレンダリングしていました：

- `currentPage.name === 'app-list'`

またこの状態のときのみアプリのクエリを有効にしていました。
そのため auth が存在していてもページの状態が `landing/login/signup` だと、`AppListPage` は `null` を返し、空の状態のテキストが決してレンダリングされませんでした。

## 実装した修正

変更ファイル：`frontend/src/features/app-list/pages/AppListPage.tsx`

1. `shouldShowAppList` を導入
2. 以下のいずれかのページで データフェッチを有効化：
   - `app-list`
   - `landing`
   - `login`
   - `signup`
3. それらの状態で AppList UI をレンダリングし、`null` を返すのをやめた

主な変更点：

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

## 結果

Smoke テストが再び green に：

```bash
npx playwright test --grep "@smoke" --project=chromium
```

結果：**2件パス**（`example.spec.ts`、`crud.spec.ts`）。

```ts
await expect(page.getByText('No apps yet. Create your first app!')).toBeVisible();
```

認証画面の導入後、ページ状態が `landing`、`login`、または `signup` のままでもアプリを認証できました。

## 根本的な原因

認証されたユーザーの場合、`App.tsx` は `AppListPage` をマウントしますが、`AppListPage` は以下の場合にのみ以前にレンダリングされました。

- `currentPage.name === 'app-list'`

また、その状態でのみアプリのクエリが有効になりました。
そのため、認証が存在してもページ状態が `landing/login/signup` のままである場合、`AppListPage` は `null` を返し、空の状態のテキストは表示されませんでした。

## 修正が実装されました

変更されたファイル: `frontend/src/features/app-list/pages/AppListPage.tsx`

1. `shouldShowAppList`を導入しました
2. ページが次のいずれかの場合にデータの取得が有効になりました。
   - `app-list`
   - `landing`
   - `login`
   - `signup`
3. `null` を返す代わりに、これらの状態の AppList UI をレンダリングしました。

キーの変更:

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

## 結果

煙テストは再び緑色になります。

```bash
npx playwright test --grep "@smoke" --project=chromium
```

結果: **2 件が合格** (`example.spec.ts`、`crud.spec.ts`)。
