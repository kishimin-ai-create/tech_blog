# 成功時のフォーム状態リセット: onSuccessコールバック内での処理

## 対象読者

- Reactでフォーム処理を実装しているエンジニア
- 非同期処理後の状態管理について学びたい開発者
- TDDを実践している人

## このArticleが対象とする範囲

このArticleでは、サインアップ成功後にフォーム入力フィールドが空にならない問題と、その解決方法を紹介します。

**対象外**: ログイン機能の詳細実装、エラーハンドリング全般

## 問題: フォーム送信成功後、入力値が残る

ユーザーがサインアップフォームに有効なメールアドレスとパスワードを入力して送信すると、API呼び出しが成功し、`onSuccess` コールバックが実行されます。しかし、その後もフォームのメールアドレスとパスワード入力フィールドに前回の入力値が残ったままになっていました。

```typescript
// 修正前のコード: onSuccess後、状態がリセットされていない
async function submitAuthRequest() {
  // ... 検証とAPI呼び出し ...
  if (!authResponse.success) {
    setError(authResponse.error)
    return
  }

  onSuccess(authResponse.data)  // ← ここで終わり、状態がリセットされない
}
```

この動作は不便です。次のサインアップ試行を促す画面に遷移する際、入力フィールドが空の状態を見ることは、ユーザー体験の観点から望ましいからです。

## 根本原因: onSuccess実行後の状態管理漏れ

`useAuthForm`フックは以下の状態を管理していました:

```typescript
const [email, setEmail] = useState('')
const [password, setPassword] = useState('')
const [error, setError] = useState<string | null>(null)
const [isSubmitting, setIsSubmitting] = useState(false)
```

API呼び出しが成功した場合、呼び出し元（SignupPageまたはLoginPage）に対して `onSuccess` コールバックが実行されます。しかし、このコールバック実行の直後に、フック側で `email` と `password` の状態を明示的にリセットしていませんでした。

つまり、フォーム状態はAPI呼び出し前の値のままになっていたのです。

## 解決策: onSuccess実行直後に状態をリセット

修正は、`onSuccess` を実行した直後に、`setEmail('')` と `setPassword('')` を呼び出す簡潔なものです。

```typescript
if (!authResponse.success) {
  setError(authResponse.error)
  return
}

onSuccess(authResponse.data)
setEmail('')                    // ← 追加
setPassword('')                 // ← 追加
```

### 修正されたコード全体

```typescript
async function submitAuthRequest() {
  if (isSubmitting) return

  if (!email.trim() || !password) {
    setError('Email and password are required')
    return
  }

  setError(null)
  setIsSubmitting(true)

  try {
    const response = await fetch(endpoint, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password }),
    })

    let responseBody: unknown = null
    const responseText = await response.text()
    if (responseText.trim().length > 0) {
      try {
        responseBody = JSON.parse(responseText) as unknown
      } catch {
        responseBody = null
      }
    }

    if (!response.ok) {
      setError(extractErrorMessage(responseBody) ?? fallbackErrorMessage)
      return
    }

    const authResponse = parseAuthResponse(responseBody)
    if (!authResponse) {
      setError(fallbackErrorMessage)
      return
    }

    if (!authResponse.success) {
      setError(authResponse.error)
      return
    }

    onSuccess(authResponse.data)
    setEmail('')
    setPassword('')
  } catch {
    setError('Unable to reach the server. Please try again.')
  } finally {
    setIsSubmitting(false)
  }
}
```

## 実装の詳細: 状態リセットのタイミング

このパターンで重要なのは、状態をリセットするタイミングです。

1. **成功応答の検証後**: `parseAuthResponse` で応答が有効であることが確認された後
2. **onSuccess実行直後**: 呼び出し元への通知を終えた直後
3. **エラーハンドリングの外**:  `finally` ブロックではなく、成功ケース専用の場所に置く

なぜ `finally` ブロックに置かないのか? 理由は、エラー発生時にはフォーム内容を保持したいからです。ユーザーは同じ入力値で再度送信を試みたいかもしれません。

```typescript
// ❌ これは間違い: エラー時もリセットされてしまう
} finally {
  setIsSubmitting(false)
  setEmail('')        // ← エラー時も実行される
  setPassword('')     // ← エラー時も実行される
}

// ✅ これが正しい: 成功時のみリセット
if (!authResponse.success) {
  setError(authResponse.error)
  return
}

onSuccess(authResponse.data)
setEmail('')          // ← 成功時のみ実行
setPassword('')       // ← 成功時のみ実行
```

## テストで検証: 送信成功後、フォームが空になる

以下のテストケースで、この動作が期待通りに動作することを確認しました:

```typescript
it('when valid registration info is submitted, then email and password input fields are cleared', async () => {
  // Arrange
  const user = userEvent.setup()
  server.use(
    http.post('/api/v1/auth/signup', () =>
      HttpResponse.json({
        success: true,
        data: {
          token: 'test-token',
          user: { id: 'user-1', email: 'test@example.com' },
        },
      }),
    ),
  )
  renderWithProviders(<SignupPage />)

  // Act
  const emailInput = screen.getByRole('textbox', { name: /email/i })
  const passwordInput = screen.getByLabelText(/password/i)
  await user.type(emailInput, 'test@example.com')
  await user.type(passwordInput, 'password123')
  await user.click(screen.getByRole('button', { name: /sign up/i }))

  // Assert
  await waitFor(() => {
    expect((emailInput as HTMLInputElement).value).toBe('')
    expect((passwordInput as HTMLInputElement).value).toBe('')
  })
})
```

このテストは以下を検証します:

1. フォームに入力を行う
2. 送信ボタンをクリック
3. API呼び出しが成功する（MSWでモック）
4. **入力フィールドの値が空になる**

## 実装パターン: 他の成功コールバックへの応用

このパターンは、他の非同期フォーム処理にも適用できます。基本的な指針:

- **ユーザーが次のアクションに進むべき場合**: 状態をリセット
  - 例: 送信成功後、同じフォームを再利用する
  - 例: 一括処理を複数回行える機能
  
- **ユーザーがフォームを修正する可能性がある場合**: 入力を保持
  - 例: バリデーションエラーの場合
  - 例: サーバーから返された編集可能な値

## 注意点: 状態リセットと副作用の順序

フック内で複数の状態更新が含まれる場合、更新順序が重要になることがあります。Reactは複数の `setState` 呼び出しをバッチ処理するため、この実装では順序は問題になりません。しかし、外部の副作用がある場合は注意が必要です。

```typescript
// 例: toastメッセージを表示し、その後フォームをリセットしたい場合
onSuccess(authResponse.data)      // 1. 親コンポーネントに通知
// showToast('Signup successful!')  // 2. 副作用（親で処理）
setEmail('')                       // 3. 状態リセット
setPassword('')
```

この順序が重要なのは、状態更新がバッチ処理されるため、`onSuccess` 呼び出しから状態リセットまでの間にコンポーネント再レンダリングが発生しないからです。

## まとめ

**この修正のポイント:**

1. **成功時には状態をリセットする**: ユーザー体験を改善し、次の操作に備える
2. **エラー時には状態を保持する**: ユーザーが再試行しやすくする
3. **タイミングを明示的に制御する**: `finally` ではなく、成功ケース専用の場所に置く
4. **テストで検証する**: 入力フィールドの値が実際に空になることをテストで確認

このパターンは小さな修正ですが、ユーザー体験に大きな影響を与えます。フォーム送信後、入力フィールドが自動的にクリアされると、アプリケーションが適切に動作していることをユーザーに示すことができます。
