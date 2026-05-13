# TDD を使用したユーザー プロファイル編集の実装

**日付:** 2026-05-11
**対象読者:** React、TypeScript、および Hono に精通したフルスタック エンジニア。フルスタック全体に TDD 規律を適用しようとしている開発者
**範囲:** この記事では、ユーザー プロファイル編集機能の完全な Red → Green TDD サイクル (`PUT /api/v1/users/:userId` バックエンド エンドポイント、`UserProfilePage` フロントエンド コンポーネント、ループを閉じる Playwright E2E テスト) について説明します。更新された電子メールまたはパスワードのデータベースへの実際の永続性についてはカバーされていません (現在の実装では、ストレージなしで値がエコー バックされます)。

---

## 背景

TDD todo アプリには、サインアップ、ログイン、ログアウトという機能する認証フローがあり、すべて Jotai の `authAtom` (`atomWithStorage`) によってサポートされていました。まだ欠けていたのは、ログインしたユーザーがサインアップ フロー全体を再度実行することなく自分の電子メールまたはパスワードを変更する方法でした。

目標は、ヘッダーの「プロファイル」ボタンの後ろにプロファイル編集画面を追加し、最初にテストによってフルスタックを駆動し、単一の実稼働ファイルが作成される前にすべてのレイヤー (検証ロジック、HTTP ハンドラー、React コンポーネント、ブラウザー フロー) をカバーし続けることです。

---

## TDD サイクル

### 赤のフェーズ — すべてのテストを最初に作成します

Red フェーズは 2 つのコミット、`f36df6a` (テスト) と `e0ae283` (緑) で構成されます。すべての運用ファイルが存在しない状態で、56 のテストすべてが 1 つのコミットで書き込まれました。レイヤーごとにグループ化された 5 つのテスト ファイル:

|ファイル |階層 |テスト |初期結果 |
|---|---|---|---|
| `backend/src/infrastructure/user-profile-validation.small.test.ts` |小（ユニット） | 19 |すべて失敗 — モジュールが見つかりません |
| `backend/src/tests/user-profile.medium.test.ts` |中（統合） | 16 |すべて失敗 — ルートは 404 を返します |
| `frontend/src/shared/navigation.small.test.ts` |小（ユニット） | 4 |すべて失敗 — `goToUserProfile` 未定義 |
| `frontend/src/features/auth/pages/UserProfilePage.medium.test.tsx` |中（統合） | 12 |すべて失敗 — モジュールが見つかりません |
| `frontend/e2e/user-profile.medium.test.ts` |中 (劇作家 E2E) | 5 |すべて失敗 — 要素が見つかりません |

56 件のテストが失敗しました。プロダクションコードはありません。それが意図された開始状態です。

Red フェーズのコミット メッセージは、生きたドキュメントとしても機能します。

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

失敗したテストでは、実装の決定が行われる前に正確な API コントラクトが定義されます。

---

### グリーン フェーズ — 合格するための最小限の実装

`e0ae283` は、56 個のテストすべてをグリーンにするために必要な製品コードを正確に追加します。 6 つのファイルが変更され、217 行が追加されました。テストで要求されたものを超える製品コードはありません。

---

## バックエンド: `PUT /api/v1/users/:userId`

### 検証 — `parseUpdateUserProfileInput`

最初の新しいファイルは `backend/src/infrastructure/user-profile-validation.ts` です。その単一のエクスポート関数 `parseUpdateUserProfileInput` は、生のリクエスト本文を検証し、型付きの判別共用体を返します。

```ts
export function parseUpdateUserProfileInput(body: unknown):
  | { success: true; email: string; currentPassword?: string; newPassword?: string }
  | { success: false; message: string }
```

19 個の単体テストによってエンコードされた検証ルール:

|ルール |行動 |
|---|---|
| `email` 文字列がないか、文字列がありません | `{ success: false, message: 'Email is required.' }` |
|電子メールが正規表現 `^[^\s@]+@[^\s@]+\.[^\s@]+$` に失敗する | `{ success: false, message: 'Please enter a valid email address.' }` |
|電子メールの先頭/末尾に空白がある |検証前にトリミングされて返されます |
| `newPassword` は存在しますが、`currentPassword` は存在しません | `{ success: false, message: 'currentPassword is required ...' }` |
| `currentPassword` は存在しますが、`newPassword` は存在しません | `{ success: false, message: 'newPassword is required ...' }` |
|どちらのパスワードも空の文字列です。 `{ success: false, message: 'Passwords cannot be empty.' }` |
|有効な電子メールのみ | `{ success: true, email }` |
|有効な電子メール + 両方のパスワード | `{ success: true, email, currentPassword, newPassword }` |

パスワードの更新は **常にペア**です。両方のフィールドを一緒に指定するか、どちらも指定しない必要があります。 1 つだけを渡すと検証エラーになります。このルールは、関数が存在する前に特定の単体テストによって取得されました。

```ts
it('when body has newPassword but no currentPassword, then returns success false', () => {
  const result = parseUpdateUserProfileInput({
    email: 'valid@example.com',
    newPassword: 'new-pass',
  })
  expect(result.success).toBe(false)
})
```

### HTTP ハンドラー — `hono-app.ts`

ルートは最小限の手順で Hono アプリに追加されます。

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

ハンドラー自体には検証ロジックは含まれておらず、完全に `parseUpdateUserProfileInput` に委任されます。統合テストでは、ステータス コード、応答本文の形状、`Content-Type` ヘッダー、および検証エラー条件の全範囲をカバーする 16 のシナリオを通じてこの構造を検証しました。すべて実際の Hono アプリ インスタンスを使用しました。

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

## フロントエンド: `UserProfilePage`

### ナビゲーション — `useNavigation` 拡張機能

ページ コンポーネントを構築する前に、ナビゲーション レイヤーに新しいエントリが必要でした。 `Page` ユニオン タイプと `frontend/src/shared/navigation.ts` の `useNavigation` フックが一緒に拡張されました。

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

この変更は 4 つの小規模な単体テストで保護されています。1 つは `goToUserProfile` が返されたオブジェクトの関数であることを確認し、3 つはそれを呼び出すと異なる開始ページから `currentPageAtom` が `{ name: 'user-profile' }` に設定されることを確認します。 4 つはすべてコードの前に書かれています。

### `UserProfilePage` コンポーネント

`frontend/src/features/auth/pages/UserProfilePage.tsx` は 128 行です。その責任:

1. `authAtom` からの **メールを事前入力**すると、ユーザーは現在のアドレスをすぐに確認できます。
2. **Conditionally render** the "現在のパスワード" field — it only appears when the user has started typing into "New Password", avoiding unnecessary UI noise.
3. **`200` 応答が成功したときに新しいメールで `authAtom`** を更新します。これにより、アプリの残りの部分には、ページをリロードせずに更新されたアドレスが表示されます。
4. **Display inline feedback** — a success message ("保存しました") or a server-provided error message, both with accessible role markup.
5. **戻るナビゲーション** — 「← 戻る」ボタンは `goToAppList()` を呼び出し、ユーザーをアプリリスト画面に戻します。

API 応答が成功した後の鍵の状態更新ブロック:

```tsx
if (json.success && json.data) {
  setAuth({
    token: auth!.token,
    user: { id: auth!.user.id, email: json.data.email },
  })
  setSuccessMessage('保存しました')
}
```

`email` フィールドのみが更新されます。既存の `token` および `user.id` は保持されます。 `useAtom(authAtom)` フックにより、`atomWithStorage` が新しい値を `localStorage` に書き込むため、更新された電子メールはページがリロードされても保持されます。

コンポーネントの中程度の統合テストは、MSW (`http.put`) を使用して API をスタブする 12 のシナリオをカバーします。

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

### アプリのルーティング — `App.tsx`

`App.tsx` の 2 つの変更により、すべてが接続されます。

1. A "プロフィール" button is rendered in the authenticated header, wired to `goToUserProfile()`.
2. 新しい条件付き分岐は、`currentPage.name === 'user-profile'` がデフォルトの認証済みレイアウトの *前* に挿入されると、`<UserProfilePage />` をレンダリングして、ビューポート全体を引き継ぎます。

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

## 劇作家の E2E テスト

`frontend/e2e/user-profile.medium.test.ts` の 5 つの E2E テストは、外側のループを閉じます。実際のバックエンドには触れずに、実際のブラウザ フロー (サインアップ フォーム → アプリ リスト → プロファイル編集) を実行します。

|テスト |検証する内容 |
|---|---|
| Profile button visible in header after login | The "プロフィール" button is in the DOM after signup |
|プロフィールボタンをクリックすると編集ページが表示されます |ヘッダーからプロフィール編集へのナビゲーションが機能します。
|現在のユーザーの電子メールが事前に入力された電子メール入力 | `authAtom` 値はコンポーネントによって正しく読み取られます。
| Saving a new email shows "保存しました" | Full form submit → API stub → success feedback loop |
| 「←戻る」ボタンでアプリ一覧に戻ります |戻るナビゲーションでプロフィール ページを離れます |

このテストでは、ログアウト E2E テストで確立されたのと同じ `page.route()` スタブ パターンを使用します。サインアップ スタブ、アプリリスト スタブ、プロファイル更新スタブはテストごとに構成されます。

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

メソッド ガード (`route.request().method() !== 'PUT'`) は、同じ URL パターンへの非 PUT リクエストを変更せずに通過させ、意図しない傍受を防ぎます。

すべてのセレクターはセマンティックな Playwright クエリを使用します。プロファイル ボタンはパターン `/profile|プロフィール|edit profile/i` と一致するため、テストは正確なラベルの文言に対して脆弱ではありません。これは、ラベル テキストがまだ洗練されている可能性がある一方で、実用的な選択です。

---

## テスト範囲の概要

|レイヤー |ファイル |階層 |カウント |
|---|---|---|---|
|バックエンドの検証 | `user-profile-validation.small.test.ts` |小 | 19 |
|バックエンド HTTP ハンドラー | `user-profile.medium.test.ts` |中 | 16 |
|フロントエンドナビゲーション | `navigation.small.test.ts` |小 | 4 |
|フロントエンドコンポーネント | `UserProfilePage.medium.test.tsx` |中 | 12 |
|ブラウザの流れ | `user-profile.medium.test.ts` (劇作家) | E2E | 5 |
| **合計** | | | **56** |

56 個のテストはすべて、製品コードの前に記述され (赤色フェーズ)、`e0ae283` の後で 56 個すべてが緑色に変わりました (緑色フェーズ)。

---

## 既知の制限事項

現在の `PUT /api/v1/users/:userId` 実装は、**送信された電子メールをデータベースに保存せずにエコーバックします**。ハンドラーは、パス パラメーターから `userId` を取得し、それを新しい電子メールとともに返しますが、保存されているユーザー レコードは更新しません。実際の永続性、つまりユーザーの電子メールを更新し、DB 内で新しいパスワードをハッシュすることは、後続のタスクです。 API コントラクトと完全なテスト スイートが用意されているため、永続性を追加するということは、テストに触れることなくハンドラーを拡張することを意味します。

---

## まとめ

|側面 |詳細 |
|---|---|
|新しい本番ファイル | `user-profile-validation.ts`、`UserProfilePage.tsx` |
|変更された本番ファイル | `hono-app.ts`、`navigation.ts`、`App.tsx` |
|新しいテスト ファイル | 5 (検証小規模、バックエンド中、ナビゲーション小規模、コンポーネント中、Playwright E2E) |
|合計テスト | 56 (すべて製品コードの前に書かれています) |
| TDD フェーズ |赤 (56 個が失敗) → 緑 (最小限の実装) |
|状態管理 | `useAtom(authAtom)` は現在の電子メールを読み取り、200 応答で更新された電子メールを書き込みます。 `atomWithStorage` は `localStorage` まで存続します。
| Password change UX | "現在のパスワード" field renders only when "New Password" is non-empty |
|既知の制限 | DB の永続性はありません。送信された値をエコーし​​ます。フォローアップとして追跡 |

本番コードの 1 行を作成する前に 56 のテストを記述するという規律は、2 つの方法で効果をもたらします。実装は、テストで定義された観察可能な動作によって完全に形成され、結果として得られるコンポーネント ツリーには、検証ロジック、HTTP ハンドラー、Jotai アトムの更新、完全なブラウザ フローなど、すべての境界で明示的なコントラクトがあり、すべて独立して検証可能です。
