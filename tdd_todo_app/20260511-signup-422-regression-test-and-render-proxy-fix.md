# 本番サインアップが「Sign up failed」になった原因調査とTDDによる回帰テスト追加

## 対象読者

- フロントエンド・バックエンドを跨ぐバグ調査の流れを学びたいエンジニア
- Render の静的サイトホスティングとプロキシルートの仕組みに興味がある方
- テストカバレッジの盲点（`response.ok` が `false` になるパスが未テスト）を経験したことがある方

---

## 問題の概要

本番環境でユーザーがサインアップフォームを送信すると、バリデーションエラーの詳細メッセージが表示されず、代わりに汎用フォールバック文言 `'Sign up failed'` が常に表示される不具合が発生した。

ローカル開発環境では再現せず、コードそのものに問題はなかった。原因は **インフラ設定の反映漏れ** にあった。

---

## 根本原因の特定

### エラーの伝播経路を追う

フロントエンドの `useAuthForm.ts` における `POST /api/v1/auth/signup` のレスポンス処理は次の分岐を持つ。

```
fetch('/api/v1/auth/signup', ...)
  ↓
responseText = await response.text()
responseBody = JSON.parse(responseText)  // 失敗すれば null のまま

if (!response.ok) {
  // 422 などの場合はここを通る
  setError(extractErrorMessage(responseBody) ?? fallbackErrorMessage)
  return
}

// 200/201 の場合はここを通る
const authResponse = parseAuthResponse(responseBody)
```

`response.ok` が `false`（ステータス 400〜599）のときは `extractErrorMessage` が呼ばれ、`true` のときは `parseAuthResponse` が呼ばれる。

### 実際に何が起きていたか

本番では `/api/v1/auth/signup` というリクエストが **静的ファイルサーバーに届いていた**。

静的ファイルサーバーはそのパスに対応するファイルを持たないため、`404 HTML` を返す。フロントエンドは HTML を `JSON.parse` しようとして失敗し、`responseBody` が `null` になる。その結果：

```
extractErrorMessage(null)
  → !isRecord(null) → return null   // null が返る

setError(null ?? 'Sign up failed')   // フォールバックが表示される
```

### なぜ静的ファイルサーバーに届いたのか

フロントエンドは相対パス `/api/*` で API を呼び出す。Render の静的サイトでは、このリクエストをバックエンドへ転送するプロキシルートを `render.yaml` に定義する必要がある。

このプロキシルートは commit `5c083de` で追加されていた：

```yaml
# render.yaml
routes:
  - type: proxy
    source: /api/*
    destination: https://tdd-todo-app-backend.onrender.com
  - type: rewrite
    source: /*
    destination: /index.html
```

しかし **Render が静的サイトを再デプロイしていなかったため**、プロキシ設定が有効になっていなかった。コードは正しいのに、デプロイが追いついていない状態だった。

---

## 発見されたテストの盲点

調査の過程で、直前のコミット `3be1928`（`parseAuthResponse` のオブジェクト形式エラーボディ対応）が追加したテストは **ステータス 200（`response.ok = true`）の経路** しかカバーしていなかったことが判明した。

バックエンドが実際に返すバリデーションエラーはステータス **422**（`response.ok = false`）であり、処理は `extractErrorMessage` を通る。このパスはテストされていなかった。

| 経路 | ステータス | 通る関数 | テスト状況（修正前） |
|---|---|---|---|
| 成功 | 201 | `parseAuthResponse` | ✅ テスト済み |
| 2xx + エラーボディ | 200 | `parseAuthResponse` | ✅ テスト済み（`3be1928` で追加） |
| バリデーションエラー | **422** | **`extractErrorMessage`** | ❌ **未テスト** |
| 非JSONレスポンス | 404 など | `extractErrorMessage` → null | ✅ テスト済み |

---

## 修正内容

### 1. フロントエンド回帰テストの追加（`SignupPage.medium.test.tsx`）

`extractErrorMessage` が実際のバックエンドエラー形状 `{ success: false, error: { code, message } }` をステータス 422 で受け取ったときに、エラーメッセージを正しく表示することを確認するテストを追加した。

```typescript
// frontend/src/features/auth/pages/SignupPage.medium.test.tsx

it('when signup API returns 422 with object-style error body, then error message from body is displayed', async () => {
  // バックエンドが実際に返す形状: 422 + { success: false, error: { code, message } }
  // response.ok が false になるため extractErrorMessage を通る経路
  const user = userEvent.setup()
  server.use(
    http.post('/api/v1/auth/signup', () =>
      HttpResponse.json(
        {
          success: false,
          error: { code: 'VALIDATION_ERROR', message: 'Password must be at least 8 characters.' },
        },
        { status: 422 },
      ),
    ),
  )
  renderWithProviders(<SignupPage />)

  await user.type(screen.getByRole('textbox', { name: /email/i }), 'test@example.com')
  await user.type(screen.getByLabelText(/password/i), 'short')
  await user.click(screen.getByRole('button', { name: /sign up/i }))

  expect(await screen.findByRole('alert')).toHaveTextContent('Password must be at least 8 characters.')
})
```

このテストが通ることで、フロントエンドが 422 レスポンスのボディからエラーメッセージを正しく取り出せることを保証する。

### 2. バックエンド回帰テストの追加（`auth.medium.test.ts`）

フロントエンドが依存するレスポンス形状をバックエンド側でも固定するため、422 のケースを 3 種類追加した。

```typescript
// backend/src/tests/integrations/infrastructure/auth.medium.test.ts

it('422: returns VALIDATION_ERROR with message when password is too short', async () => {
  const res = await request('POST', '/api/v1/auth/signup', {
    email: 'test@example.com',
    password: 'short',
  })
  expect(res.status).toBe(422)
  const json = await res.json() as { success: boolean; error: { code: string; message: string } }
  expect(json.success).toBe(false)
  expect(json.error.code).toBe('VALIDATION_ERROR')
  expect(typeof json.error.message).toBe('string')
  expect(json.error.message.length).toBeGreaterThan(0)
})

// 同様に: 不正なメール形式、ボディなしの計3ケース
```

これにより「フロントエンドが期待する `{ success: false, error: { code, message } }` 形状をバックエンドが確実に返す」という契約がテストで担保される。

### 3. コミットプッシュによる Render 自動再デプロイのトリガー

テストを追加したコミット `a34ea9e` をプッシュすることで、Render の auto-deploy が走り静的サイトが再ビルド・再デプロイされた。これによりプロキシルートが有効になり、本番の `/api/*` リクエストがバックエンドへ正しく転送されるようになった。

---

## `extractErrorMessage` の実装

参考として、問題の核心にある `extractErrorMessage` の実装を示す。

```typescript
// frontend/src/features/auth/hooks/useAuthForm.ts

const extractErrorMessage = (value: unknown): string | null => {
  if (!isRecord(value)) return null

  // 文字列形式: { error: "メッセージ" }
  if (typeof value.error === 'string' && value.error.trim().length > 0) {
    return value.error
  }

  // オブジェクト形式: { error: { code: "...", message: "メッセージ" } }
  if (isRecord(value.error) && typeof value.error.message === 'string') {
    const message = value.error.message.trim()
    return message.length > 0 ? message : null
  }

  return null
}
```

`value` が `null` や非オブジェクトの場合（HTMLが渡された場合など）は最初の `!isRecord(value)` で `null` を返す。この `null` がフォールバックメッセージの表示につながる。

---

## 注意点

### 「コードに問題なし」でも本番で壊れることがある

今回の不具合の本質は、コードは正しく動作していたが **インフラ設定の反映が遅れた** ことにある。ローカルでは Vite の開発サーバーが `/api/*` をプロキシしていたため、問題が顕在化しなかった。

### テストは「実際に起きるステータスコード」で書く

`parseAuthResponse`（2xx 経路）のテストだけでは不十分だった。バックエンドが 422 を返す仕様であれば、フロントエンドのテストも 422 のモックで書く必要がある。`response.ok` の真偽で処理が分岐する実装では、両方のパスを明示的にテストすることが重要である。

### フロントエンド・バックエンド間の「契約」をテストで固定する

バックエンドのレスポンス形状が変わるとフロントエンドのエラー表示が壊れる。今回追加したバックエンドのテストは、フロントエンドが依存する形状 `{ success: false, error: { code, message } }` が変更されたときに CI で検出できるセーフガードとなっている。

---

## まとめ

| 観点 | 内容 |
|---|---|
| **直接原因** | Render 静的サイトのプロキシルートが未デプロイ → API リクエストが静的ファイルサーバーに届き 404 HTML を返した |
| **エラー伝播** | HTML → JSON.parse 失敗 → `responseBody = null` → `extractErrorMessage(null)` = null → フォールバック表示 |
| **テストの盲点** | 422（`response.ok = false`）+ オブジェクト形式エラーボディの経路が未テストだった |
| **修正** | フロントエンドに 422 テスト、バックエンドに 422 × 3 テストを追加。プッシュで Render 再デプロイをトリガー |
| **教訓** | `response.ok` で分岐する実装は両パスをテストする。インフラ変更はデプロイ完了まで追跡する |
