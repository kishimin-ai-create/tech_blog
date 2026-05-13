# Reactのkey属性を使ったフォーム状態の独立性確保

## 対象読者

- React初級者～中級者（特に条件付きレンダリングを使う人）
- フォーム管理の落とし穴を理解したい開発者
- `key` 属性の正確な役割を学びたい人

## このArticleが対象とする範囲

このArticleでは、複数のページをモーダルのように条件付きレンダリングする際に、フォーム状態が前のページから継承される問題と、`key` 属性による解決方法を説明します。

**対象外**: 実装パターン全般、ルーティングの詳細

## 問題: ページ切り替え時、フォーム入力が保持される

ユーザーがサインアップページでメールアドレスとパスワードを入力した後、ログインページに切り替えると、前のページで入力した値がそのまま表示されていました。

具体的には:

1. ユーザーが **サインアップページ** でメール「signup@example.com」とパスワード「signuppass」を入力
2. ページ切り替えボタンをクリックして **ログインページ** に遷移
3. **ログインページのフォームに「signup@example.com」と「signuppass」が残っている**（期待値: 空の状態）

これはセキュリティとUXの両面で問題があります。一つのアプリケーション内で複数のフォームが共存するなら、それぞれが独立した状態を持つべきです。

## 根本原因: Reactのkey属性の欠落

修正前のコードは以下のようなものでした:

```typescript
// frontend/src/App.tsx (修正前)
if (!auth) {
  if (currentPage.name === 'signup') return <SignupPage />     // ← keyなし
  if (currentPage.name === 'login') return <LoginPage />       // ← keyなし
  return <LandingPage />
}
```

### なぜこれが問題なのか?

ReactのJSXを見ると、`<SignupPage />` と `<LoginPage />` は異なるコンポーネント型です。しかし、Reactの`key`がない場合、同じツリーの位置に異なるコンポーネントをレンダリングするとき、以下の動作になります:

1. **初回レンダリング**: `currentPage.name === 'signup'` → `<SignupPage />` が作成
   - `useAuthForm` フックが実行され、`useState` は初期値 `''` で初期化
   - ユーザーが入力を行う

2. **ページ切り替え**: `currentPage.name === 'login'` に変更
   - **Reactはツリー構造から判断して、コンポーネント型が違うため、SignupPageをアンマウント**
   - **新しく `<LoginPage />` をマウント**

一見すると問題ないように思えますが、重要な点は:

- `<SignupPage />` と `<LoginPage />` は別の関数コンポーネント
- しかし、どちらも `useAuthForm` フックを呼び出している
- **`key` がないため、Reactはこれらを「同じツリー位置の異なるコンポーネント」と認識**
- その結果、Reactが最適化を試みた際に、前のコンポーネントの状態が引き継がれることがある

より正確には: `key` がなく、JSX要素が同じ「位置」にある場合、Reactは要素の型だけで判断するため、型が同じフックライブラリを使用していると、状態がリセットされずに残る可能性があります。

実装を見ると、`useAuthForm` は両方のページで使用されており、フック呼び出し順序も同じです。このため、`useState` の状態値がコンポーネント切り替え時に保持されていたのです。

## 解決策: key属性でコンポーネント識別

修正は簡潔です。`SignupPage` と `LoginPage` に異なる `key` を付与します:

```typescript
// frontend/src/App.tsx (修正後)
if (!auth) {
  if (currentPage.name === 'signup') return <SignupPage key="signup" />    // ← keyを追加
  if (currentPage.name === 'login') return <LoginPage key="login" />       // ← keyを追加
  return <LandingPage />
}
```

### key属性の役割

`key` は以下の指示をReactに与えます:

- **`key="signup"`**: 「このツリー位置には、キー 'signup' でマークされた要素を置く」
- **`key="login"`**: 「このツリー位置には、キー 'login' でマークされた要素を置く」

その結果:

1. **初回**: `currentPage.name === 'signup'`
   - `<SignupPage key="signup" />` がツリーに追加される
   - `useAuthForm` フック内の `useState` が初期化: `email = ''`, `password = ''`

2. **ページ切り替え**: `currentPage.name === 'login'`
   - Reactは: 「キー 'signup' の要素がなくなった」と判断
   - Reactは: 「キー 'login' の要素が新しく出現した」と判断
   - **キー 'signup' のコンポーネントを完全にアンマウント** (状態はクリア)
   - **キー 'login' のコンポーネントを新規マウント** (新しい状態を初期化)

3. **再度サインアップに切り替え**: `currentPage.name === 'signup'`
   - キー 'signup' のコンポーネントが再度マウント
   - **新しい状態インスタンスが作成される** (前の値は残っていない)

## 実装詳細: key属性の仕組み

### key がない場合の動作 (問題のあるケース)

```
レンダリング1: currentPage = 'signup'
┌─────────────────┐
│ <SignupPage />  │  ← useState(email='')
│ (コンポーネント)  │  ← useState(password='')
└─────────────────┘

ユーザー入力後:
┌─────────────────┐
│ <SignupPage />  │  ← email='signup@example.com'
│ (コンポーネント)  │  ← password='signuppass'
└─────────────────┘

レンダリング2: currentPage = 'login' ← keyなし
┌─────────────────┐
│ <LoginPage />   │  ← React: 「型は違うが同じ位置」と判断
│ (コンポーネント)  │  ← 状態がリセットされずに残る可能性
└─────────────────┘

実際の挙動: 前のページの値が表示されてしまう
```

### key がある場合の動作 (修正後)

```
レンダリング1: currentPage = 'signup'
┌──────────────────────────┐
│ <SignupPage key="signup" │  ← useState(email='')
│ (コンポーネント)          │  ← useState(password='')
└──────────────────────────┘

ユーザー入力後:
┌──────────────────────────┐
│ <SignupPage key="signup" │  ← email='signup@example.com'
│ (コンポーネント)          │  ← password='signuppass'
└──────────────────────────┘

レンダリング2: currentPage = 'login' ← keyあり
┌──────────────────────────┐
│ <LoginPage key="login"   │  ← React: 「キーが異なる」と判断
│ (コンポーネント)          │  ← key='signup'をアンマウント
└──────────────────────────┘       ← key='login'を新規マウント (状態初期化)

実際の挙動: ログインページが空の状態で表示される
```

## テストで検証: フォーム独立性の確認

以下のテストケースで、ページ切り替え時のフォーム独立性を検証します:

```typescript
it('when switched between Signup and Login pages, then form fields remain independent', async () => {
  // Arrange
  const user = userEvent.setup()
  const store = createStore()
  store.set(currentPageAtom, { name: 'signup' })

  // Act: SignupPageをレンダリング
  renderWithProviders(<App />, { store })
  
  // SignupPageのフォームに入力
  const signupEmailInput = screen.getByRole('textbox', { name: /email/i })
  const signupPasswordInput = screen.getByLabelText(/password/i)
  
  await user.type(signupEmailInput, 'signup@example.com')
  await user.type(signupPasswordInput, 'signuppassword123')
  
  expect(signupEmailInput).toHaveValue('signup@example.com')
  expect(signupPasswordInput).toHaveValue('signuppassword123')
  
  // LoginPageに切り替え
  store.set(currentPageAtom, { name: 'login' })
  
  // LoginPageのフォームが空の状態であることを確認
  await waitFor(() => {
    const loginEmailInput = screen.getByRole('textbox', { name: /email/i })
    const loginPasswordInput = screen.getByLabelText(/password/i)
    expect(loginEmailInput).toHaveValue('')           // ← 空であることを期待
    expect(loginPasswordInput).toHaveValue('')        // ← 空であることを期待
  })
  
  // LoginPageのフォームに別の入力を行う
  const loginEmailInput = screen.getByRole('textbox', { name: /email/i })
  const loginPasswordInput = screen.getByLabelText(/password/i)
  
  await user.type(loginEmailInput, 'login@example.com')
  await user.type(loginPasswordInput, 'loginpassword123')
  
  expect(loginEmailInput).toHaveValue('login@example.com')
  expect(loginPasswordInput).toHaveValue('loginpassword123')
  
  // SignupPageに戻す
  store.set(currentPageAtom, { name: 'signup' })
  
  // SignupPageが空の状態で表示されることを確認
  await waitFor(() => {
    const backToSignupEmailInput = screen.getByRole('textbox', { name: /email/i })
    const backToSignupPasswordInput = screen.getByLabelText(/password/i)
    expect(backToSignupEmailInput).toHaveValue('')        // ← 前の入力は保持されない
    expect(backToSignupPasswordInput).toHaveValue('')     // ← 前の入力は保持されない
  })
})
```

このテストの重要なポイント:

1. **3回のページ切り替えサイクル**: signup → login → signup
2. **各ページで異なる入力値を使用**: signup用と login用で別の値
3. **各切り替え後、フォームが空になることを確認**: 前のページの値は保持されない

## key属性の一般的なガイドライン

### ✅ key を付与すべき場合

1. **条件付きレンダリング内の複数コンポーネント**
   ```typescript
   if (condition) return <ComponentA key="a" />
   else return <ComponentB key="b" />
   ```

2. **同じ親要素内で異なる子要素を条件で切り替える**
   ```typescript
   {showForm1 && <Form1 key="form1" />}
   {showForm2 && <Form2 key="form2" />}
   ```

3. **状態を保持したくない場合**
   - ページナビゲーション
   - モーダルの切り替え
   - 条件付きフィルター表示

### ❌ key を付与してはいけない場合

1. **リスト内のアイテム（別の方法を使う）**
   ```typescript
   // ❌ 悪い例
   {items.map((item, index) => <Item key={index} {...item} />)}
   
   // ✅ 良い例
   {items.map(item => <Item key={item.id} {...item} />)}
   ```

2. **安定したユニークIDがない場合** はリスト専用ではない他の手段を検討

## React内部の状態管理: useState と key の関係

`key` と `useState` の関係を理解することが重要です:

```typescript
function LoginPage() {
  // これらの状態はコンポーネントインスタンスに紐付いている
  const [email, setEmail] = useState('')
  const [password, setPassword] = useState('')
  // ...
}

function SignupPage() {
  // 別のコンポーネント型なので、独立した状態インスタンス
  const [email, setEmail] = useState('')
  const [password, setPassword] = useState('')
  // ...
}
```

**key がない場合**: Reactはコンポーネントの「型」だけで判断するため、同じツリー位置で異なる型が交互に現れると、状態がリセットされ忘れる可能性があります。

**key がある場合**: Reactはコンポーネントの「型+key」で判断するため、`key` が変わるとそのコンポーネントインスタンスは完全に新しく作り直されます。

## 実装パターン: 拡張可能な例

より複雑なシナリオでも同じパターンが適用できます:

```typescript
function App() {
  const [auth] = useAtom(authAtom)
  const [currentPage] = useAtom(currentPageAtom)

  if (!auth) {
    // 複数の認証ページを条件付きで表示
    if (currentPage.name === 'signup') 
      return <SignupPage key="signup" />
    if (currentPage.name === 'login') 
      return <LoginPage key="login" />
    if (currentPage.name === 'forgot-password') 
      return <ForgotPasswordPage key="forgot-password" />
    return <LandingPage />
  }

  // 認証済みユーザーのページでも同様
  if (currentPage.name === 'detail')
    return <DetailPage id={currentPage.id} key={`detail-${currentPage.id}`} />
  if (currentPage.name === 'edit')
    return <EditPage id={currentPage.id} key={`edit-${currentPage.id}`} />
  
  return <ListPage />
}
```

注意: `key` にはコンポーネントの用途を識別できるユニークな値を使用します。単なる文字列の場合と、IDを含める場合を使い分けます。

## まとめ

**このバグと修正のポイント:**

1. **問題**: 条件付きレンダリングで `key` がないと、Reactは同じツリー位置にあるコンポーネントの状態をリセットしないことがある

2. **解決**: 異なるコンポーネントには異なる `key` を付与し、Reactが明示的に「このコンポーネントはアンマウントされた」と認識できるようにする

3. **効果**: 各ページが独立した状態を持つようになり、ページ切り替え時に前の入力値が残らなくなる

4. **テスト**: 複数ページの独立性を検証することで、バグの再発を防ぐ

`key` 属性は一見すると小さな属性ですが、Reactの状態管理の根底に関わる重要な仕組みです。特に複数のフォームを条件付きで切り替えるような場合は、`key` の存在が大きな違いを生みます。
