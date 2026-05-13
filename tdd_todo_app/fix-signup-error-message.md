# 修正：サインアップが常に「Sign up failed」を表示し実際のエラーメッセージが表示されない問題

**日付:** 2026-05-11  
**技術スタック:** React · TypeScript · Hono (Node.js) · Render.com · MSW · Vitest
**変更ファイル:**
- `frontend/src/features/auth/hooks/useAuthForm.ts`
- `frontend/src/features/auth/pages/SignupPage.medium.test.tsx`

---

## エラーの概要

無効なデータでサインアップしようとすると、UI は常にハードコードされたフォールバックメッセージ **「Sign up failed」** を表示していました——バックエンドが実際に何を報告したかに関わらず。

期待される動作は **「Password must be at least 8 characters.」** のような具体的で対処可能なメッセージを表示することでした。しかし実際には、すべてのバリデーションの失敗が同じ汎用文字列にサイレントに折り畳まれ、ユーザーは何を修正すべきか全くわかりませんでした。

---

## 原因

### バックエンドのエラー形状

Hono バックエンドは 422 バリデーションエラーに対して以下の JSON ボディを返します：

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Password must be at least 8 characters."
  }
}
```

`error` は文字列ではなく**オブジェクト**です。

### Render.com でのプロキシの問題

フロントエンドは `/api/*` をバックエンドウェブサービスにプロキシする Render.com の静的サイトとしてデプロイされています。特定の条件下で、Render のプロキシレイヤーがバックエンドの `4xx` レスポンスを `200`（または別の `2xx`）に変換します。これにより `response.ok` が `true` に評価されてしまいますが、ボディには `{ success: false, error: { ... } }` が含まれています。

### `parseAuthResponse` の壊れたガード

`useAuthForm.ts` には、`response.ok` が `true` であることが確認されてから JSON ボディをデコードする純粋なパーサー関数 `parseAuthResponse` が含まれていました。**修正前**の関連部分：

```typescript
// ❌ 修正前（元の67-68行目）
if (typeof value.error !== 'string') return null

return { success: false, error: value.error }
```

意図は「文字列のエラーがなければ中断する」でした。実際には、プロキシが 422 を 200 に変換すると、ボディには `value.error` がオブジェクトとして設定された状態で届きます——`typeof value.error !== 'string'` ガードが `true` になり、関数は即座に `null` を返しました。

### フォールバックが実際のメッセージを隠す

呼び出し元（`submitAuthRequest`）では、`parseAuthResponse` から `null` が返されると直接以下に落ちます：

```typescript
if (!authResponse) {
  setError(fallbackErrorMessage)   // "Sign up failed" ——常に
  return
}
```

つまり実際のバリデーションメッセージはパーサーの境界で破棄され、不透明なフォールバックに置き換えられ、UI に届くことは一切ありませんでした。

---

## 修正

修正は `parseAuthResponse` を拡張して、文字列以外をすべて回復不能なパース失敗として扱うのではなく、**両方**のエラー形状——プレーンな文字列と `message` フィールドを持つオブジェクト——を処理するようにしました。

**修正後**（`useAuthForm.ts`）：

```typescript
// ✅ 修正後
if (typeof value.error === 'string') {
  return { success: false, error: value.error }
}

if (isRecord(value.error) && typeof value.error.message === 'string') {
  return { success: false, error: value.error.message }
}

return null
```

ヘルパー `isRecord`（すでにファイルに存在）は値が非 null オブジェクトかどうかを確認します：

```typescript
const isRecord = (value: unknown): value is Record<string, unknown> =>
  typeof value === 'object' && value !== null
```

この変更により：

| `value.error` の形状 | 動作 |
|---|---|
| `"some string"` | 直接抽出——修正前と変わらず |
| `{ code: "...", message: "..." }` | `message` が抽出される——**新たに対応** |
| その他 | `null` を返す（フォールバックが表示）——修正前と変わらず |

`null` パスは引き続き到達可能ですが、バックエンドの文書化されたバリデーションエラー構造ではなく、真に不正なボディに対してのみです。

---

## テストカバレッジ（TDD サイクル）

修正は厳密な RED → GREEN サイクルに従いました。

### RED — 失敗するテストを先に

`SignupPage.medium.test.tsx` に本番のシナリオを再現する新しい medium テストを追加しました。MSW がサインアップリクエストをインターセプトし、オブジェクト形式のエラーボディを持つ `200` を返します：

```typescript
it('when signup API returns 200 with object-style error body, then error message from body is displayed', async () => {
  // Arrange — バックエンドの 4xx を 200 に変換するプロキシをシミュレート（JSON ボディは保持）
  server.use(
    http.post('/api/v1/auth/signup', () =>
      HttpResponse.json(
        {
          success: false,
          error: { code: 'VALIDATION_ERROR', message: 'Password must be at least 8 characters.' },
        },
        { status: 200 },
      ),
    ),
  )
  renderWithProviders(<SignupPage />)

  await user.type(screen.getByRole('textbox', { name: /email/i }), 'test@example.com')
  await user.type(screen.getByLabelText(/password/i), 'password123')
  await user.click(screen.getByRole('button', { name: /sign up/i }))

  // Assert
  expect(await screen.findByRole('alert')).toHaveTextContent('Password must be at least 8 characters.')
})
```

修正前は、このテストが失敗していました：アラートに期待したメッセージではなく `"Sign up failed"` が含まれていました。

### 追加カバレッジ：201 ステータスのハッピーパス

`parseAuthResponse` がバックエンドの `201 Created` レスポンス（これも `2xx` なので同じ `response.ok` ブランチを通る）を正しく処理することを確認するため、2番目のテストが追加されました（`Error Handling` describe ブロック内に配置されているが、成功の境界をテストする）：

```typescript
it('when signup succeeds with 201 status, then authAtom is populated and page navigates to app-list', async () => {
  server.use(
    http.post('/api/v1/auth/signup', () =>
      HttpResponse.json(
        {
          success: true,
          data: { token: 'test-token', user: { id: 'user-1', email: 'test@example.com' } },
        },
        { status: 201 },
      ),
    ),
  )
  // ... authAtom と currentPageAtom が正しく更新されたことをアサート
})
```

### GREEN — すべてのテストがパス

`parseAuthResponse` の変更後、両方の新規テストと既存の100以上のテストすべてがパスしました：

```
npm run lint && npm run typecheck && npm run test:small && npm run test:medium
# → 102テスト、全パス
```

---

## まとめ

| | 詳細 |
|---|---|
| **症状** | バックエンドの理由に関わらず、すべてのサインアップ失敗で `"Sign up failed"` が表示された |
| **根本原因** | `parseAuthResponse` が `typeof value.error !== 'string'` をガードとして使用していたため、Hono からのオブジェクト形式のエラーに対して `null` を返し、呼び出し元がハードコードされたフォールバックに落ちた |
| **トリガー条件** | Render.com プロキシがバックエンドの `422` を `200` に変換し、`response.ok === true` になってオブジェクト形式のボディが `parseAuthResponse` に届いた |
| **修正** | `{ code, message }` 形状のエラーに対する明示的なブランチを追加；`value.error.message` が正しく抽出されるようになった |
| **変更行数** | `useAuthForm.ts` の4行（`-2 / +6`） |
| **追加テスト** | 2つの新規 medium テスト——バグシナリオ用と `201` ハッピーパス用 |

核心となる教訓はおなじみのものです：**パーサーからの `null` 返却はサイレントな破棄です。** パーサーが生のネットワークボディと UI の間の単一チョークポイントである場合、過度に狭い型ガードは欠落した catch ブロックと同じくらい効果的に正当なデータを抑制できます。修正は真に不正なレスポンスに対してフォールバックパスを維持しながら、バックエンドの文書化されたエラー構造が明示的に処理されるようにします。

---

## エラーの概要

ユーザーが運用環境で無効なデータを使用してサインアップしようとすると、UI が常に表示されます
ハードコードされたフォールバック メッセージ **「サインアップに失敗しました」** — バックエンドの内容に関係なく
実際に報告されました。

期待される動作は、次のような具体的で実行可能なメッセージを表示することでした。
**「パスワードは少なくとも 8 文字である必要があります。」** 代わりに、検証が失敗するたびに
同じ汎用文字列に静かに折りたたまれ、ユーザーには何の情報も与えられません。
修正します。

---

## 原因

### バックエンドのエラー形状

Hono バックエンドは、422 検証エラーで次の JSON 本文を返します。

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Password must be at least 8 characters."
  }
}
```

`error` は文字列ではなく **オブジェクト** です。

### Render.com のプロキシの複雑さ

フロントエンドは、`/api/*` をプロキシする Render.com 静的サイトとしてデプロイされます。
バックエンド Web サービス。特定の条件下では、レンダーのプロキシ レイヤーは
バックエンド `4xx` 応答を `200` (または別の `2xx`) に送信します。これは `response.ok` を意味します
ボディに `{ success: false, error: { ... } }` が含まれている場合でも、`true` と評価されます。

### `parseAuthResponse` の壊れたガード

`useAuthForm.ts` には、デコードされた純粋なパーサー関数 `parseAuthResponse` が含まれていました。
`response.ok` が `true` であることが確認された後の JSON ボディ。該当セクション
**修正前**:

```typescript
// ❌ Before fix (lines 67-68 in original)
if (typeof value.error !== 'string') return null

return { success: false, error: value.error }
```

その目的は、「文字列エラーがない場合は救済する」というものでした。実際には、プロキシが
422 を 200 に変換すると、`value.error` がオブジェクトに設定されたボディが到着しました。
`typeof value.error !== 'string'` ガードは `true` であり、関数は返されました
すぐに`null`。

### フォールバックは実際のメッセージをマスクします

呼び出し元 (`submitAuthRequest`) に戻ると、`parseAuthResponse` からの `null` の結果が返されます。
直接的には次のようになりました。

```typescript
if (!authResponse) {
  setError(fallbackErrorMessage)   // "Sign up failed" — always
  return
}
```

そのため、実際の検証メッセージはパーサー境界で破棄され、次のメッセージに置き換えられました。
不透明なフォールバックであり、UI には到達しませんでした。

---

## Fix

この修正により、**両方**のエラー形状 (プレーン) を処理できるように `parseAuthResponse` が拡張されました。
文字列と `message` フィールドを持つオブジェクト — 文字列以外のものを扱う代わりに
回復不能な解析失敗として。

**修正後** (`useAuthForm.ts`):

```typescript
// ✅ After fix
if (typeof value.error === 'string') {
  return { success: false, error: value.error }
}

if (isRecord(value.error) && typeof value.error.message === 'string') {
  return { success: false, error: value.error.message }
}

return null
```

ヘルパー `isRecord` (ファイル内にすでに存在) は、値が
null 以外のオブジェクト:

```typescript
const isRecord = (value: unknown): value is Record<string, unknown> =>
  typeof value === 'object' && value !== null
```

この変更により:

| `value.error` 形状 |行動 |
|---|---|
| `"some string"` |直接抽出 — 以前から変更なし |
| `{ code: "...", message: "..." }` | `message` が抽出されます — **新しくサポートされました** |
|その他 | `null` を返します (フォールバックを表示) — 変更なし |

`null` パスはまだ到達可能ですが、真に奇形なボディに対してのみ到達可能であり、
バックエンドの文書化された検証エラー構造。

---

## テストカバレッジ (TDD サイクル)

修正は厳密な RED → GREEN サイクルに従いました。

### 赤 - 最初にテストに失敗しました

正確な結果を再現するために、新しい中程度のテストが `SignupPage.medium.test.tsx` に追加されました。
制作シナリオ。 MSW はサインアップ要求をインターセプトし、次のメッセージを含む `200` を返します。
オブジェクト形式のエラー本文:

```typescript
it('when signup API returns 200 with object-style error body, then error message from body is displayed', async () => {
  // Arrange — simulates a proxy that converts backend 4xx to 200, preserving the JSON body
  server.use(
    http.post('/api/v1/auth/signup', () =>
      HttpResponse.json(
        {
          success: false,
          error: { code: 'VALIDATION_ERROR', message: 'Password must be at least 8 characters.' },
        },
        { status: 200 },
      ),
    ),
  )
  renderWithProviders(<SignupPage />)

  await user.type(screen.getByRole('textbox', { name: /email/i }), 'test@example.com')
  await user.type(screen.getByLabelText(/password/i), 'password123')
  await user.click(screen.getByRole('button', { name: /sign up/i }))

  // Assert
  expect(await screen.findByRole('alert')).toHaveTextContent('Password must be at least 8 characters.')
})
```

修正前は、このテストは失敗しました。アラートには、代わりに `"Sign up failed"` が含まれていました。
予想されるメッセージ。

### 追加対象範囲: 201 ステータス ハッピー パス

2 番目のテストが追加されました (`Error Handling` 記述ブロックに配置されましたが、
成功境界をテストして)、`parseAuthResponse` が正しく処理することを確認します。
バックエンドの `201 Created` 応答 - これも `2xx` であるため、通過します
同じ `response.ok` ブランチ:

```typescript
it('when signup succeeds with 201 status, then authAtom is populated and page navigates to app-list', async () => {
  server.use(
    http.post('/api/v1/auth/signup', () =>
      HttpResponse.json(
        {
          success: true,
          data: { token: 'test-token', user: { id: 'user-1', email: 'test@example.com' } },
        },
        { status: 201 },
      ),
    ),
  )
  // ... assert authAtom and currentPageAtom updated correctly
})
```

### 緑 — すべてのテストに合格

`parseAuthResponse` の変更後、新しいテストと 100 以上の既存のテストの両方
合格した：

```
npm run lint && npm run typecheck && npm run test:small && npm run test:medium
# → 102 tests, all pass
```

---

## まとめ

| |詳細 |
|---|---|
| **症状** |バックエンドの理由に関係なく、すべてのサインアップ失敗で `"Sign up failed"` が表示されました。
| **根本原因** | `parseAuthResponse` は `typeof value.error !== 'string'` をガードとして使用したため、Hono からのオブジェクト スタイル エラーに対して `null` を返し、呼び出し元はハードコードされたフォールバックに落ちました。
| **発動条件** | Render.com プロキシがバックエンド `422` を `200` に変換するため、`response.ok === true` とオブジェクト スタイルのボディは `parseAuthResponse` に到達しました。
| **修正** | `{ code, message }` 形状のエラーに対する明示的な分岐を追加しました。 `value.error.message` は正しく抽出されるようになりました。
| **行が変更されました** | `useAuthForm.ts` (`-2 / +6`) の 4 行 |
| **テストを追加** | 2 つの新しい中程度のテスト — 1 つはバグ シナリオ用、もう 1 つは `201` ハッピー パス用 |

核となる教訓はよく知られたものです: **パーサーからの `null` 戻り値はサイレントです。
Discard.** パーサーが生のネットワーク本体とネットワーク本体の間の単一のチョークポイントである場合
UI、過度に狭い型ガードは、正規のデータを効果的に抑制する可能性があります。
キャッチブロックがありません。この修正により、真に不正な形式の応答に対するフォールバック パスが維持されます。
バックエンドの文書化されたエラー構造が明示的に処理されることを保証しながら。
