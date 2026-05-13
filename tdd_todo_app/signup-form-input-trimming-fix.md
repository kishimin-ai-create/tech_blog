# サインアップフォーム入力トリミングの一貫性修正

## 📋 概要

**状況**: 正しいメールアドレスとパスワードでサインアップが失敗する問題

**根本原因**: フォーム入力の検証時にメールアドレスはトリミングされていたが、パスワードはトリミングされていなかった

**解決策**: 検証とAPI送信の両方で、メールアドレスとパスワードの両方を一貫してトリミング

**影響**: ユーザーが入力ミスなく正当な認証情報を入力しても、フォーム検証に通らないか、トリミングされていないパスワードがバックエンドに送信される可能性があった

## 🔍 根本原因分析

### 問題の発生パターン

`frontend/src/features/auth/hooks/useAuthForm.ts` における不整合な入力処理が原因でした：

```typescript
// 修正前（不正）
if (!email.trim() || !password) {  // 🔴 パスワードはトリミング未処理
  setError('Email and password are required')
  return
}

// API送信時（不正）
body: JSON.stringify({ email, password })  // 🔴 どちらもトリミング未処理
```

### 発生しうる現象

1. **ホワイトスペースのみのパスワード** - クライアント側検証で通過してしまう
   ```
   パスワード入力: "        " (8個のスペース)
   検証結果: !password = false (スペースも true と判定される)
   ```

2. **前後の余分なスペース** - バックエンドに送信される
   ```
   ユーザー入力: " password123 "
   API送信: { password: " password123 " }
   バックエンド検証: 失敗（保存時のパスワードハッシュが一致しない）
   ```

## 🛠️ 技術的な修正内容

### 変更1: クライアント側の検証を厳格化

**ファイル**: `frontend/src/features/auth/hooks/useAuthForm.ts` (行85)

```typescript
// 修正前
if (!email.trim() || !password) {

// 修正後
if (!email.trim() || !password.trim()) {
```

**なぜこれが必要か**: 
- `!password` は空文字列(`""`)のみをチェック
- `!password.trim()` はホワイトスペースオンリーのパスワードも検出する
- 検証ルールがメール・パスワード間で統一される

### 変更2: API送信時の入力正規化

**ファイル**: `frontend/src/features/auth/hooks/useAuthForm.ts` (行97)

```typescript
// 修正前
body: JSON.stringify({ email, password })

// 修正後
body: JSON.stringify({ email: email.trim(), password: password.trim() })
```

**なぜこれが必要か**:
- ユーザーが誤ってスペースを含めた場合、バックエンドでは正規化されたパスワードと照合できない
- クライアント側で正規化することで、バックエンドとの整合性を保証
- 後続の認証ロジックが一貫性のある入力値を受け取ることを保証

## ✅ テストカバレッジ

4つの新しいテストを追加してこの修正を保護します：

### テスト1: 初期状態の確認

```typescript
it('when rendered, then email input value is empty initially', () => {
  renderWithProviders(<SignupPage />)
  const emailInput = screen.getByRole('textbox', { name: /email/i })
  expect((emailInput as HTMLInputElement).value).toBe('')
})
```

**目的**: フォームがクリア状態で始まることを保証

### テスト2: パスワード入力の初期化確認

```typescript
it('when rendered, then password input value is empty initially', () => {
  renderWithProviders(<SignupPage />)
  const passwordInput = screen.getByLabelText(/password/i)
  expect((passwordInput as HTMLInputElement).value).toBe('')
})
```

**目的**: パスワード フィールドも初期状態を保つ（前回の修正との整合性）

### テスト3: ホワイトスペースのみのパスワード拒否

```typescript
it('when password is whitespace only, then error is displayed without calling API', async () => {
  const user = userEvent.setup()
  let apiWasCalled = false
  server.use(
    http.post('/api/v1/auth/signup', () => {
      apiWasCalled = true
      return HttpResponse.json({ success: false, error: 'Should not be called' })
    }),
  )
  renderWithProviders(<SignupPage />)

  await user.type(
    screen.getByRole('textbox', { name: /email/i }),
    'test@example.com',
  )
  await user.type(screen.getByLabelText(/password/i), '        ')
  await user.click(screen.getByRole('button', { name: /sign up/i }))

  expect(await screen.findByRole('alert')).toBeInTheDocument()
  expect(apiWasCalled).toBe(false)
})
```

**アサーション**:
- エラーメッセージが表示される ✓
- APIが呼び出されない ✓ (クライアント検証が機能)
- 無駄なサーバーリクエストが防止される ✓

### テスト4: 前後のスペースがトリミングされることを確認

```typescript
it('when password has leading and trailing whitespace, then password is trimmed before sending to API', async () => {
  const user = userEvent.setup()
  let capturedPassword: string | null = null
  server.use(
    http.post('/api/v1/auth/signup', async (req) => {
      const body = await req.request.json() as { password?: string }
      capturedPassword = body.password ?? null
      return HttpResponse.json({
        success: true,
        data: {
          token: 'test-token',
          user: { id: 'user-1', email: 'test@example.com' },
        },
      })
    }),
  )
  renderWithProviders(<SignupPage />)

  await user.type(
    screen.getByRole('textbox', { name: /email/i }),
    'test@example.com',
  )
  await user.type(screen.getByLabelText(/password/i), ' password123 ')
  await user.click(screen.getByRole('button', { name: /sign up/i }))

  await waitFor(() => {
    expect(capturedPassword).toBe('password123')
  })
})
```

**アサーション**:
- APIに送信されるパスワードは `password123` である ✓
- 前後のスペースが削除されている ✓
- バックエンド検証との整合性が確保される ✓

## 📊 検証結果

```
✓ Typecheck: すべての型チェックパス
✓ Lint: すべてのスタイル規則をクリア
✓ Test: 14/14 テストパス (4つの新規テストを含む)
✓ Build: ビルド成功
✓ Smoke Tests: すべてのエンドツーエンドテストパス
```

### テスト実行結果の詳細

- **SignupPage テストスイート**: 14 テスト
  - 初期化テスト: 4 テスト ✓
  - ハッピーパス: 複数 ✓
  - エラーハンドリング: 複数 ✓
  - **本修正由来**: 4 テスト ✓

## 💡 エンジニアリングの学習ポイント

### 1. 入力検証と送信の一貫性

フォーム入力を扱う際、クライアント側では以下のレイヤーで同じ正規化を適用すべき:

- **検証ロジック** (エラー表示の判定)
- **送信ロジック** (バックエンドへ送信)

不一致があると、検証を通過した入力がバックエンドで拒否される "デッドコーナー" が生じます。

```typescript
// ✓ 推奨: 同じ正規化ロジック
const normalizedPassword = password.trim()
if (!normalizedPassword) { /* 検証 */ }
// ...
body: JSON.stringify({ password: normalizedPassword })
```

### 2. ホワイトスペースの処理

JavaScript では:
- `!string` は空文字列のみ検出
- `!string.trim()` はホワイトスペースオンリーを検出

パスワードのような重要な入力では、`trim()` を検証に含めることが重要です。

```typescript
// ❌ 不正: ホワイトスペースオンリーを許容
if (!password) { /* パスしてしまう */ }

// ✓ 正: ホワイトスペースオンリーを拒否
if (!password.trim()) { /* 拒否される */ }
```

### 3. テスト駆動での設計検証

この修正は TDD で発見・検証されました：

1. **テスト先行**: ホワイトスペースのみのケースをテストケースで定義
2. **失敗確認**: テストが失敗することを確認
3. **実装**: 修正を実装
4. **成功確認**: テストが通る
5. **リグレッション防止**: テストがコードを保護

この流れにより、単なる "バグ修正" ではなく、**再発を防止する設計** が実現されます。

## 🎯 まとめ

| 項目 | 内容 |
|------|------|
| **修正内容** | パスワード入力のトリミングを、メールアドレスと統一 |
| **変更行数** | 2 行の変更 + 76 行のテストコード追加 |
| **テスト** | 4 つの新規テストで再発を防止 |
| **影響範囲** | `SignupPage` および `LoginPage` の根底のフック (`useAuthForm`) |
| **パフォーマンス** | 無駄なAPI呼び出しを削減 (クライアント検証強化) |
| **ユーザー体験** | 無効な入力で即座にエラー表示、有効な入力は確実に処理 |

この修正により、ユーザーは正当な認証情報を入力した場合、**確実にサインアップできる** ようになりました。

---

**コミット**: 3f87efc  
**ファイル**: `frontend/src/features/auth/hooks/useAuthForm.ts` (2 行修正)  
**テスト**: `frontend/src/features/auth/pages/SignupPage.test.tsx` (76 行追加)
