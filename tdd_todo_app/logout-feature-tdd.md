# React + Jotai の TDD を使用したログアウト機能の実装

**日付:** 2025-07-25
**対象読者:** React と TypeScript に精通したフロントエンド エンジニア。 TDD ワークフローを学ぶ開発者
**範囲:** この記事では、Jotai 状態管理に基づくログアウト機能のレッド→グリーン→リファクタリング→コード レビューのサイクル全体をカバーします。サーバー側のトークンの取り消し (フォローアップ タスクとして認識されます) については説明しません。

---

## 背景

TDD todo アプリには、Jotai アトムを中心に構築されたサインアップとログインのフローがすでにありました。ユーザーが認証されると、トークンとユーザー プロファイルが `authAtom` (`atomWithStorage` アトム) に保存され、アクティブ ページが `currentPageAtom` で追跡されます。欠けていたのは、ユーザーが _out_ を取得する方法でした。ログアウト フックもログアウト ボタンも、セッションのティアダウンをトリガーする認証済みレイアウトには何もありませんでした。

目標は単純で、認証状態をクリアし、ログイン ページにリダイレクトし、ページがリロードされてもそのクリア状態を維持します。しかし、そこに到達するまでのパスは、最初にテスト、次に実装というように意図的に系統立てて行われました。

---

## TDD サイクル

### 赤のフェーズ - 失敗したテストの書き込み

実稼働コードを 1 行記述する前に、次の 3 つのテスト ファイルが作成されました。

|ファイル |サイズ階層 |目的 |
|---|---|---|
| `useLogout.small.test.ts` |小（ユニット） |フック API 表面、原子状態遷移、境界ケース |
| `LogoutButton.small.test.tsx` |小（ユニット） | `useLogout` をモック化したレンダリングとクリックのインタラクション |
| `LogoutButton.medium.test.tsx` |中（統合） |実際の Jotai ストアを使用した完全な `App` レンダリング |

この時点では、`useLogout.ts` と `LogoutButton.tsx` は存在しませんでした。すべてのテストはモジュール解決エラーで失敗しました。それは予想通りでした。失敗したテストが契約を定義します。

フックの単体テストでは、次の 4 つのシナリオを事前にカバーしました。

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

統合テストでは、外部から観察可能な UI の動作を検証しました。

```ts
it('when the logout button is clicked, then authAtom is set to null', ...)
it('when the logout button is clicked, then currentPageAtom becomes { name: "login" }', ...)
it('when the logout button is clicked, then App renders the LoginPage with an email input', ...)
it('when the logout button is clicked, then the "ログアウト" button is no longer in the document', ...)
```

合計 17 のテスト。全部赤い。

---

### グリーン フェーズ — 最小限の実装

テストによって定義されたコントラクトにより、実装はコンパクトであることが判明しました。

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

ここで注目すべき点が 2 つあります。

1. **`useAtom` の代わりに `useSetAtom`** — フックは書き込むだけで済みます。現在の認証値を読み取ることはありません。 `useSetAtom` は、コンポーネントが関係のない状態変更をサブスクライブすることを回避し、意図を明示的にします。

2. **`atomWithStorage` および `null`** — `authAtom` は `atomWithStorage<AuthState | null>('auth', null)` として定義されます。 `setAuth(null)` が呼び出されると、Jotai の `atomWithStorage` 実装は、`'auth'` キーの下で `null` を `localStorage` に書き込みます。次のページの読み込み時に、アトムは `null` (保存された値) に初期化されるため、ブラウザーの更新後でもセッションは適切に終了します。

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

**`App.tsx` — 認証されたレイアウトへのボタンの追加**

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

`LogoutButton` は、`App` の認証されたブランチ内で無条件にレンダリングされます。 `auth` が `null` の場合はレンダリングされないため、認証されていないユーザーに表示されるリスクはありません。

17 個のテストすべてに合格しました。緑。

---

### リファクタリング フェーズ - 品質の向上

テストがグリーンになると、動作を変えることなくコードの品質に焦点が移りました。

- プロジェクトの `jsdoc/require-jsdoc` ESLint ルール (`publicOnly: true`、`FunctionDeclaration: true`) を満たすために、`useLogout` と `LogoutButton` の両方に JSDoc コメントを追加しました。
- 残りの UI と一貫したスタイルを実現するために、Tailwind クラスを `LogoutButton` に適用しました。
- ボタンが `<form>` 内に配置された場合に誤ってフォームが送信されるのを防ぐために、`type="button"` を `<button>` 要素に追加しました。

テストは全体を通して緑色のままでした。

---

## テスト構造の詳細

### 小規模なテスト — ユニットの分離

`LogoutButton` の単体テストは、完全に `useLogout` をモックします。

```ts
vi.mock('../hooks/useLogout', () => ({
  useLogout: vi.fn(),
}))

// eslint-disable-next-line import/order -- placed here for readability; vi.mock is hoisted by Vitest regardless
import { useLogout } from '../hooks/useLogout'
```

`vi.mock()` 呼び出しは、Vitest のコンパイラー変換によってモジュールの先頭にホイストされるため、ソース順序での `import` ステートメントの位置は、モックがアクティブかどうかに影響しません。 `eslint-disable-next-line` コメントには、この事実が関連する行に直接記載されています。

`useLogout` の単体テストでは、テストごとに分離された実際の Jotai ストアが使用されます。

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

各テストは独自の `createStore()` インスタンスを取得します。 `atomWithStorage` は副作用として `localStorage` との間で読み取りと書き込みを行うため、`localStorage.clear()` は各テストの前に実行されます。クリアしないと、前のテストのアトムが次のテストに流れ込む可能性があります。

### 中規模のテスト — 実店舗を介した統合

統合テストでは、事前にシードされたストアを使用して完全な `App` コンポーネントをレンダリングし、アトム レベルではなく UI レベルでアサーションを作成します (ただし、両方とも検証されています)。

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

MSW サーバーは `GET /api/v1/apps` エンドポイントをスタブするため、`AppListPage` は実際のバックエンドなしでレンダリングされます。これにより、実際のコンポーネント ツリーを実行しながら、統合テストの密閉性が維持されます。

---

## コードレビューの結果

実装とリファクタリングの後、構造化されたコード レビューにより 4 つの発見結果が明らかになりました。 3 つはすぐに修正されました。 1件は追跡追跡調査としてリスクを許容された。

### 修正: `onClick` から矢印ラッパーが削除されました

初期実装では以下が使用されました。

```tsx
onClick={() => { logout() }}
```

`logout` は引数を取らない `() => void` 関数であるため、これによりレンダリングのたびに新しい関数参照が作成され、何のメリットもありません。修正は直接代入です。

```tsx
onClick={logout}
```

ここには、将来の微妙な問題もあります。`logout` が非同期にされた場合、アロー ラッパーは返された `Promise` を黙って破棄します。参照を直接渡すことで、意図が明確になります。

この修正に加えて、対応するテスト アサーションが `.toHaveBeenCalledWith()` (引数ゼロ チェック) から `.toHaveBeenCalled()` に更新されました。直接代入では、React は `SyntheticEvent` をハンドラーに渡すため、引数がゼロかどうかのチェックは失敗します。存在チェックによって実際の意図がキャプチャされます。

### 修正: `import/order` ルールのスコープが正しく設定されました

`eslint.config.js` の以前のバージョンでは、`vi.mock` / `import` 順序パターンに対応するために、すべてのテスト ファイルにわたって `import/order` をグローバルに無効にしていました。レビューではこれが広すぎると特定されました。実際の例外が 1 つのファイル内の 1 つのインポート行であった場合、増大するテスト スイート全体でルールが沈黙されました。

この修正により、グローバル オーバーライドが、ルールが起動される正確な行でターゲットを絞ったインライン無効化に置き換えられました。

```ts
// eslint-disable-next-line import/order -- placed here for readability; vi.mock is hoisted by Vitest regardless
import { useLogout } from '../hooks/useLogout'
```

インポート順序規則は、他のすべてのテスト ファイルに適用されるようになりました。

### 修正: 冗長な「アクセス可能なロール」テストが削除されました

テスト スイートには当初、`LogoutButton.small.test.tsx` に 2 つの記述ブロックが含まれていました。

```ts
// Test A
expect(screen.getByRole('button', { name: 'ログアウト' })).toBeInTheDocument()

// Test B (redundant)
expect(screen.getByRole('button')).toBeInTheDocument()
```

テスト B はテスト A の厳密なサブセットです。ボタンが正しいラベルで正しく表示される場合、ロール チェックはすでに暗黙的に行われています。要素が `<button>` ではなく `<div>` としてレンダリングされる場合、両方のテストは同時に失敗します。テスト B はゼロの独立信号を提供するため、削除されました。

### リスクの受け入れ: サーバー側のトークンの取り消しはありません

`useLogout` はクライアント側の状態のみをクリアします。バックエンドには `POST /api/v1/auth/logout` エンドポイントが存在せず、何も追加されませんでした。 `localStorage['auth']` に保存されている JWT は、自然期限が切れるまでサーバーによって受け入れられ続けます。「ログアウト」をクリックしても、バックエンドで JWT が無効になることはありません。

ログアウト前にトークンが (XSS またはネットワーク インターセプト経由で) 抽出された場合、クライアント側のログアウトでは保護が提供されません。認められている軽減策は次のとおりです。

1. サーバー側のトークン ブラックリストを使用して `POST /api/v1/auth/logout` を実装します。運用前のフォローアップ タスクとして追跡されます。
2. トークンの有効期限を短く (15 分以内) にして、その間の攻撃ウィンドウを制限します。
3. ステートレス JWT が必要な場合は、サーバー管理の失効リストを使用してリフレッシュ トークンを発行します。

これはアーキテクチャ レベルの制約であり、単一の機能ブランチで完全に解決できるものではありません。リスクは文書化され、緩和策として短期間で受け入れられます。

---

## まとめ

|側面 |詳細 |
|---|---|
|新しいファイル | `useLogout.ts`、`LogoutButton.tsx`、3 つのテスト ファイル |
|変更されたファイル | `App.tsx`、`eslint.config.js` |
|テスト数 | 17 テスト (7 フック ユニット、5 コンポーネント ユニット、5 統合) |
|状態管理 |常体 `useSetAtom`; `atomWithStorage` localStorage を null クリアします。
| TDD フェーズ |赤 (17 個の失敗したテスト) → 緑 (最小限の実装) → リファクタリング (ドキュメント、スタイル、タイプ セーフティ) → コード レビュー (4 つの発見、3 つの修正) |
|既知の制限 |サーバー側のトークンの取り消しはありません。 JWT の有効期限が短いことで軽減される |

実装は設計上最小限です。2 つの運用ファイル、明確なフックとコンポーネントの分離、および両方のユニットを分離して実際のコンポーネント ツリーを通じて一緒に実行するテスト スイートです。コード レビュー パスでは、実際の問題 (ひっそりと広がる可能性のある lint スコープのバグ、冗長なテスト ケース、微妙なレンダリング パフォーマンスの問題) が検出され、最終コードは最初のパスのグリーン状態よりも大幅に改善されました。
