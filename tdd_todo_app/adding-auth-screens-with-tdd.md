# TDD で認証画面を追加する：Red → Green → Refactor → Review

**対象読者：** TDD を実際の UI 機能——認証フォーム、共有フック、Jotai を使った状態管理——に適用する様子を見たいフロントエンド開発者。

---

## 概要

TDD Todo App にはかつて認証ゲートがありませんでした——どの訪問者もそのままアプリに入れました。この記事では、3つの認証画面（Landing、Login、Signup）を追加し、永続的な認証状態を組み込み、構造化されたコードレビューで発見された5つの実際のバグを修正するために使われた完全な TDD サイクルを説明します——最終的に171のテストがパスする状態で。

スタック技術：React、TypeScript、Jotai、Tailwind CSS、Vitest、React Testing Library、MSW。

---

## Red フェーズ：テストを先に

本番コードを1行も書く前に、3つの新しいページと `App.tsx` のルーティング動作のテストを書きました。すべて失敗しました——それが目的です。

テストが事前に指定した主なこと：

- `LandingPage` が**Login**ボタンと**Sign Up**ボタンをレンダリングする
- `LoginPage` が `POST /api/v1/auth/login` に資格情報を POST し、成功時に `authAtom` を設定して `app-list` にナビゲートする
- `SignupPage` が `POST /api/v1/auth/signup` に対して同じことをする
- `App.tsx` が `authAtom` が `null` のときに認証画面を表示し、認証済み時にアプリ画面を表示する

MSW ハンドラーが API レスポンスをモックして、テストを高速に保ち実際のバックエンドから分離しました。

---

## Green フェーズ：最小限の実装

失敗するテストを仕様として、最小限の本番コードを書きました：

**認証状態** — Jotai の `atomWithStorage` 1つがトークンとユーザーを `localStorage` に永続化：

```ts
// frontend/src/shared/auth.ts
import { atomWithStorage } from 'jotai/utils'

export type AuthState = {
  token: string
  user: { id: string; email: string }
}

export const authAtom = atomWithStorage<AuthState | null>('auth', null)
```

`atomWithStorage` を使ったのは意図的な選択です：E2E テストは、実際のログインフローを必要とせずにアプリが読み込む前に `localStorage.setItem('auth', JSON.stringify({ token, user }))` を呼び出すことで認証状態をシードできます。

**App.tsx での認証ゲーティング** — `useEffect` なしの純粋な条件付きレンダリング：

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

認証エンドポイントが OpenAPI の仕様に含まれていなかったため、`fetch` を直接使用しました（axios や orval なし）——実装をシンプルで依存なしに保っています。

---

## Refactor フェーズ：`useAuthForm` の抽出

`LoginPage` と `SignupPage` はほぼ同一のフォームロジック——同じ `email` 状態、`password` 状態、`error` 状態、`fetch` 呼び出し——を持っていました。Refactor フェーズでは共有の `useAuthForm` フックを抽出することでこの重複を排除しました：

```ts
// frontend/src/features/auth/hooks/useAuthForm.ts
export function useAuthForm({ endpoint, fallbackErrorMessage, onSuccess }: UseAuthFormOptions) {
  const [email, setEmail] = useState('')
  const [password, setPassword] = useState('')
  const [error, setError] = useState<string | null>(null)
  const [isSubmitting, setIsSubmitting] = useState(false)

  async function submitAuthRequest() {
    if (isSubmitting) return                          // 再入ガード
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

各ページはシンプルなコンシューマーになりました——エンドポイントを設定して `onSuccess` を定義するだけ：

```tsx
const { email, setEmail, password, setPassword, error, isSubmitting, handleSubmit } = useAuthForm({
  endpoint: '/api/v1/auth/login',
  fallbackErrorMessage: 'Login failed',
  onSuccess: (auth) => { setAuth(auth); goToAppList() },
})
```

---

## Review フェーズ：5つのバグを発見して修正

テストが Green になった後に構造化されたコードレビューを実施しました。5つの問題が浮上しました：

| 優先度 | 問題 | 修正 |
|---|---|---|
| P1 | **二重送信** — fetch が進行中にガードなし | `isSubmitting` 状態を追加；送信中はボタンを無効化してラベルを変更（「Logging in…」） |
| P1 | **ナビゲーションボタンに `type="button"` がない** — HTML のデフォルトは `type="submit"` | `<form>` 外のすべてのナビゲーションボタンに明示的な `type="button"` を追加 |
| P2 | **`currentPageAtom` の初期値が `'app-list'`** — 最初の読み込み時に atom とレンダリングされるページが同期していなかった | 状態を正直にするために初期値を `{ name: 'landing' }` に変更 |
| P2 | **クライアント側バリデーションなし** — 空の認証情報がサーバーに届いていた | `fetch` 呼び出し前の `submitAuthRequest` にトリム/空ガードを追加 |
| P3 | **ネットワークエラーブランチがテストされていない** — `catch` パスのカバレッジがゼロ | `LoginPage.test.tsx` と `SignupPage.test.tsx` の両方に MSW `HttpResponse.error()` テストを追加 |

特に P1 のバグは通常のハッピーパスの手動テストでは見えなかったでしょう——特定のタイミングやブラウザの条件下でのみ現れます。Refactor フェーズでもロジックを集約することで `isSubmitting` 修正がサイレントに導入されていて、2ヶ所ではなく1ヶ所にガードを追加するのが簡単になっていました。

`localStorage` / XSS リスク（P2）は認識されましたが先送りにされました：仕様が E2E シードの互換性のために `atomWithStorage` を明示的に要求しているため、`httpOnly` Cookie への移行にはバックエンドの調整が必要で、既知のリスクとして追跡されています。

---

## 最終状態

- **3つの新規ページ**：`LandingPage`、`LoginPage`、`SignupPage`
- **1つの共有フック**：`useAuthForm` — 両方のフォームページで使用、ロジックの重複ゼロ
- **認証状態**：`atomWithStorage` 経由で `localStorage` に永続化、E2E テストから読み取り可能
- **ルーティング**：`App.tsx` での純粋な条件付きレンダリング、エフェクトベースのリダイレクトなし
- **テスト**：16のテストファイルで171がパス

---

## 教訓

1. **ページの前にテストを書く。** 実装の詳細に気を取られる前に、完全な API コントラクト（どの状態が設定されるか、どのナビゲーションが起きるか）を定義せざるを得ません。
2. **Refactor フェーズはデザインが行われる場所。** 2つのページが存在した後に `useAuthForm` を抽出することで、適切な抽象化が明らかになりました——早まった推測ではなく。
3. **Review フェーズはテストが見落としたものを捉える。** 二重送信バグと `type="button"` の省略は、通常のテストでは両方サイレントでしたが、敵対的な条件下では実際のリスクでした。
4. **状態の一貫性は、動作が正しく見えるときでも重要。** `currentPageAtom` が `'app-list'` で初期化されることは可視のバグを引き起こしませんでしたが、それは嘘をついている atom でした——嘘の状態は本物のバグになるのを待っている技術的負債です。

---

## 概要

以前の TDD Todo アプリには認証ゲートがなく、訪問者はアプリに直接アクセスしていました。この投稿では、3 つの認証画面 (ランディング、ログイン、サインアップ) を追加し、永続的な認証状態を接続し、構造化コード レビュー中に発見された 5 つの実際のバグを修正するために使用される完全な TDD サイクルを順を追って説明します。最後に 171 のテストに合格します。

技術スタック: React、TypeScript、Jotai、Tailwind CSS、Vitest、React テスト ライブラリ、MSW。

---

## レッドフェーズ: まずはテスト

実稼働コードの 1 行を作成する前に、3 つの新しいページと `App.tsx` ルーティング動作のテストが作成されました。全員が失敗したが、それが目標だった。

テストで事前に指定された重要な点:

- `LandingPage` は、**ログイン** ボタンと **サインアップ** ボタンをレンダリングします
- `LoginPage` は、資格情報を `POST /api/v1/auth/login` にポストし、成功すると、`authAtom` にデータを入力して、`app-list` に移動します。
- `SignupPage` は `POST /api/v1/auth/signup` に対して同じことを行います
- `App.tsx` は、`authAtom` が `null` の場合の認証画面と、認証された場合のアプリ画面を表示します。

MSW ハンドラーは API 応答を模擬するため、テストは高速に行われ、実際のバックエンドから分離されます。

---

## グリーンフェーズ: 最小限の実装

失敗したテストを仕様として使用して、最小限の実稼働コードが作成されました。

**認証状態** — Jotai からの単一の `atomWithStorage` は、トークンとユーザーを `localStorage` に永続化します。

```ts
// frontend/src/shared/auth.ts
import { atomWithStorage } from 'jotai/utils'

export type AuthState = {
  token: string
  user: { id: string; email: string }
}

export const authAtom = atomWithStorage<AuthState | null>('auth', null)
```

`atomWithStorage` の使用は意図的な選択でした。E2E テストでは、実際のログイン フローを必要とせず、アプリが読み込まれる前に `localStorage.setItem('auth', JSON.stringify({ token, user }))` を呼び出すことで認証状態をシードできます。

**App.tsx の認証ゲート** — 純粋な条件付きレンダリング、`useEffect` なし:

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

認証エンドポイントが OpenAPI 仕様に含まれていなかったため、`fetch` が直接使用され (axios や orval は使用されませんでした)、実装がシンプルで依存関係がなくなりました。

---

## リファクタリング フェーズ: `useAuthForm` の抽出

`LoginPage` と `SignupPage` は、ほぼ同じ形式のロジック、つまり、同じ `email` 状態、`password` 状態、`error` 状態、および `fetch` 呼び出しを持っていました。リファクタリング フェーズでは、共有 `useAuthForm` フックを抽出することで、この重複を排除しました。

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

各ページはシン コンシューマになります。エンドポイントを構成して `onSuccess` を定義するだけです。

```tsx
const { email, setEmail, password, setPassword, error, isSubmitting, handleSubmit } = useAuthForm({
  endpoint: '/api/v1/auth/login',
  fallbackErrorMessage: 'Login failed',
  onSuccess: (auth) => { setAuth(auth); goToAppList() },
})
```

---

## レビュー段階: 5 つのバグが見つかり修正されました

テストが成功した後に構造化コードレビューが実行されました。次の 5 つの問題が明らかになりました。

|優先順位 |問題 |修正 |
|---|---|---|
| P1 | **ダブルサブミット** — フェッチの実行中はガードなし | `isSubmitting` 状態を追加しました。送信中にボタンが無効になり、ラベルが変更されました (「ログイン中…」)。
| P1 | **ナビゲーション ボタンが `type="button"` がない** — HTML のデフォルトは `type="submit"` | `<form>` の外側のすべてのナビゲーション ボタンに明示的な `type="button"` を追加しました。
| P2 | **`currentPageAtom` の初期値は `'app-list'`** — アトムとレンダリングされたページは最初のロード時に同期していませんでした。状態を正直にするために初期値を `{ name: 'landing' }` に変更しました。
| P2 | **クライアント側の検証なし** - 空の資格情報がサーバーに到達しました。 `fetch` 呼び出しの前に `submitAuthRequest` にトリム/空ガードを追加しました。
| P3 | **ネットワーク エラー ブランチはテストされていません** - `catch` パスのカバレッジはゼロです | MSW `HttpResponse.error()` テストを `LoginPage.test.tsx` と `SignupPage.test.tsx` の両方に追加しました。

特に P1 のバグは、ハッピー パスの手動テストでは見えなかったでしょう。特定のタイミングまたはブラウザの条件下でのみ表面化します。リファクタリング フェーズでは、ロジックを集中管理することによって `isSubmitting` 修正もサイレントに導入され、ガードを 2 か所ではなく 1 か所に簡単に追加できるようになりました。

`localStorage` / XSS リスク (P2) は認識されましたが、延期されました。仕様では、E2E シード互換性のために `atomWithStorage` が明示的に必要とされているため、`httpOnly` Cookie への移行にはバックエンドの調整が必要であり、既知のリスクとして追跡されます。

---

## 最終状態

- **3 つの新しいページ**: `LandingPage`、`LoginPage`、`SignupPage`
- **1 共有フック**: `useAuthForm` — 両方のフォーム ページで使用され、ロジックの重複はありません
- **認証状態**: `atomWithStorage` 経由で `localStorage` に保持され、E2E テストで読み取り可能
- **ルーティング**: `App.tsx` での純粋な条件付きレンダリング、エフェクトベースのリダイレクトなし
- **テスト**: 16 のテスト ファイルで 171 が合格

---

## テイクアウト

1. **ページの前にテストを作成します。** 実装の詳細に気をとられる前に、完全な API コントラクト (どのような状態が設定されるか、どのようなナビゲーションが行われるか) を定義する必要があります。
2. **リファクタリング段階では設計が行われます。** 2 ページが存在した後で `useAuthForm` を抽出すると、時期尚早な推測ではなく、適切な抽象化が明らかになりました。
3. **レビュー段階で、テストで見逃した部分が見つかります。** 二重送信のバグと `type="button"` の省略はどちらも、通常のテストでは発生しませんでしたが、敵対的な状況では実際のリスクが発生します。
4. **動作が正しく見える場合でも、状態の一貫性は重要です。** `currentPageAtom` を `'app-list'` に初期化しても、目に見えるバグは発生しませんでしたが、それは横たわったアトムでした。そして、横たわった状態は、実際のバグになるのを待っている技術的負債です。
