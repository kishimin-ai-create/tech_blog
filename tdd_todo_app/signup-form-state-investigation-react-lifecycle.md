# バグ調査で学ぶ React のコンポーネントライフサイクルと key プロップの本当の役割

## 対象読者

- React + TypeScript でフロントエンド開発をしている中級者
- バグ報告を受けたときに何から調べればよいか知りたいエンジニア
- `key` プロップが「なぜ」効くのかを正確に理解したい開発者
- 回帰テストの書き方を学びたい人

---

## バグ報告

> サインアップ画面でフォームに値を入力してからログイン画面に移動し、再びサインアップ画面に戻ると、入力した値がそのまま残っている。

フォームのリセット忘れ、Jotai の atom への誤った状態保存、あるいはルーティングの実装ミス——原因としていくつかの仮説が考えられる。今回は実際にコードを追いながら、「何が起きているか」「何が起きていないか」を順番に確認した調査プロセスと、その中で見えてきた React のライフサイクルに関する重要な知識を記録する。

---

## 調査の進め方

### Step 1：フォーム状態の持ち方を確認する

まず「状態がどこに住んでいるか」を確認した。フォーム入力値が複数ページをまたいで保持されるとしたら、グローバルな状態管理（このプロジェクトでは Jotai）を使っているはずだ。

`useAuthForm.ts` を開くと、フォームに関わるすべての状態がローカルな `useState` であることがわかった：

```ts
// frontend/src/features/auth/hooks/useAuthForm.ts

export function useAuthForm({ endpoint, fallbackErrorMessage, onSuccess }: UseAuthFormOptions) {
  const [email, setEmail]           = useState('')
  const [password, setPassword]     = useState('')
  const [error, setError]           = useState<string | null>(null)
  const [isSubmitting, setIsSubmitting] = useState(false)
  // ...
}
```

Jotai の atom は一切使っていない。`email` も `password` も、このフックを呼び出したコンポーネントインスタンスにしか存在しない。  
→ **グローバル状態への誤保存という仮説は否定された。**

### Step 2：ルーティング実装を確認する

次に `App.tsx` を見た。Jotai の `currentPageAtom` で現在のページを管理し、「早期 return」パターンで条件分岐している：

```tsx
// frontend/src/App.tsx

function App() {
  const [auth] = useAtom(authAtom)
  const [currentPage] = useAtom(currentPageAtom)

  if (!auth) {
    if (currentPage.name === 'signup') return <SignupPage key="signup" />
    if (currentPage.name === 'login')  return <LoginPage  key="login"  />
    return <LandingPage />
  }
  // ...
}
```

ここで重要なことが 2 つある。

**① `SignupPage` と `LoginPage` は異なるコンポーネント型である**  
React は JSX ツリーの同じ位置に異なるコンポーネント型が現れると、前のコンポーネントを完全にアンマウントして新しいコンポーネントをマウントする。これは `key` の有無に関わらず発生する。  
→ ページ遷移のたびに `useAuthForm` 内の `useState` はゼロから初期化される。

**② `key` プロップが付いている（`key="signup"`, `key="login"`）**  
こちらは「防御的な」実装だ。その理由は後述する。

### Step 3：コミット履歴を確認する

```
ca13d16  fix: ensure form state independence between signup and login pages by adding keys
```

このコミットがすでに存在していた。コミットメッセージと差分を確認すると、`<SignupPage />` と `<LoginPage />` に `key` プロップを追加し、あわせてフォーム独立性を検証する回帰テストを追加している。

### Step 4：テストを確認する

`App.test.tsx` の 66〜116 行目に、まさにバグ報告が指摘するシナリオを網羅したテストがある：

```ts
it('when switched between Signup and Login pages, then form fields remain independent', async () => {
  // SignupPage に入力 → LoginPage に切り替え → フォームが空 → SignupPage に戻る → フォームが空
})
```

180 件のテストがすべてパスしていることも確認済み。

**結論：このバグはすでに修正されている。**

---

## なぜバグが発生していたか（再現可能な根本原因）

修正内容を読み解くにあたって、「コンポーネント型が違えば key は不要では？」という疑問が生じる。これを正確に理解するために、React の差分計算アルゴリズムの基本を整理する。

### React がコンポーネントのアイデンティティを決める 2 つの軸

React は再レンダリング時に「同じコンポーネントインスタンスを使い続けるか、新しく作り直すか」を次の 2 つで判断する：

| 判断軸 | 変化したときの挙動 |
|---|---|
| **コンポーネント型**（関数・クラス） | アンマウント → 新規マウント（`useState` リセット） |
| **`key` プロップ** | アンマウント → 新規マウント（`useState` リセット） |

どちらか一方が変化しただけで、React は既存のファイバーを破棄して新しいインスタンスを生成する。

### 「型が違えば key は不要」は正しいが、それだけではない

今回の `App.tsx` では `SignupPage` と `LoginPage` は別の型なので、型変化だけでアンマウントは起きる。しかし `key` を加えることで得られる追加の保護がある：

```
もし将来、SignupPage と LoginPage が同一コンポーネント（例: AuthPage）にリファクタリングされた場合、
key がなければ React は同じコンポーネントインスタンスを再利用し、useState の値が前のページから引き継がれる。
key="signup" / key="login" があれば、型が同じでも確実にアンマウント → マウントが発生する。
```

コミットメッセージには「missing keys」が原因と書かれているが、より正確には：
- **一次的な修正機構**：型が違うことによるアンマウント（これだけでも動作する）
- **防御的な実装**：`key` プロップによる明示的なインスタンス分離（将来の型統合リファクタリングに対する保護）

---

## React コンポーネントライフサイクルと useState の関係

以下の 3 パターンの挙動を整理しておく。

### パターン A：型が変わる（key なし）

```tsx
// currentPage が 'signup' → 'login' に変わる
if (currentPage.name === 'signup') return <SignupPage />   // アンマウント
if (currentPage.name === 'login')  return <LoginPage />    // 新規マウント・useState リセット
```

異なる型が同じツリー位置に現れるため、React は前のインスタンスを破棄する。`useState` は初期値から始まる。フォームはリセットされる。

### パターン B：型が同じ・key が変わる

```tsx
// 将来のリファクタリング後のイメージ
if (currentPage.name === 'signup') return <AuthPage key="signup" mode="signup" />
if (currentPage.name === 'login')  return <AuthPage key="login"  mode="login"  />
```

型は同じ `AuthPage` だが、`key` が変わるため React は前のインスタンスを破棄して新しく作る。`useState` はリセットされる。

### パターン C：型が同じ・key もない（フォーム値が残るケース）

```tsx
// ❌ バグが起きる実装
if (currentPage.name === 'signup') return <AuthPage mode="signup" />
if (currentPage.name === 'login')  return <AuthPage mode="login"  />
```

型も `key` も変わらないので、React は同じコンポーネントインスタンスを再利用する。`useState` の値はそのまま保持される。`mode` プロップだけが更新されるため、フォームに前のページの入力値が残る。

---

## 早期 return パターンがルーティングにもたらす利点

`App.tsx` が採用している「条件付き早期 return」パターンは、ルーター設計において見落とされがちな利点を持つ。

```tsx
if (!auth) {
  if (currentPage.name === 'signup') return <SignupPage key="signup" />
  if (currentPage.name === 'login')  return <LoginPage  key="login"  />
  return <LandingPage />
}
```

**JSX の return が分岐ごとに独立しているため、React のツリー上の「位置」が変わる。**

これに対して、三項演算子や `&&` を使った条件付きレンダリングでは、コンポーネントがツリー上の「同じ位置」に存在し続けるため、型が同じであれば再利用が発生しうる：

```tsx
// ⚠️ 型が同じなら key なしでは危険
return currentPage.name === 'signup'
  ? <AuthPage mode="signup" />
  : <AuthPage mode="login" />
```

早期 return パターンを使うか、三項・論理演算子を使う場合は必ず `key` を付ける、という方針が安全だ。

---

## ナビゲーションによる状態リセットの回帰テスト

このバグで重要なのは「修正した」だけでなく「回帰を防ぐテストが存在する」点だ。ナビゲーション起因の状態リセットを検証するテストは次の構造を持つ：

```ts
it('when switched between Signup and Login pages, then form fields remain independent', async () => {
  // Arrange: SignupPage を表示
  const user = userEvent.setup()
  const store = createStore()
  store.set(currentPageAtom, { name: 'signup' })
  renderWithProviders(<App />, { store })

  // Act 1: SignupPage にデータを入力して確認
  const signupEmail = screen.getByRole('textbox', { name: /email/i })
  await user.type(signupEmail, 'signup@example.com')
  expect(signupEmail).toHaveValue('signup@example.com')

  // Act 2: LoginPage に切り替え
  store.set(currentPageAtom, { name: 'login' })

  // Assert: LoginPage のフォームが空であること
  await waitFor(() => {
    expect(screen.getByRole('textbox', { name: /email/i })).toHaveValue('')
  })

  // Act 3: LoginPage にデータを入力
  await user.type(screen.getByRole('textbox', { name: /email/i }), 'login@example.com')

  // Act 4: SignupPage に戻る
  store.set(currentPageAtom, { name: 'signup' })

  // Assert: SignupPage のフォームが空であること（前の入力が残っていない）
  await waitFor(() => {
    expect(screen.getByRole('textbox', { name: /email/i })).toHaveValue('')
  })
})
```

このテストが「回帰テスト」として機能するためのポイントは 3 つある：

1. **往復する（signup → login → signup）**：片道だけではなく、ナビゲーションの往復が状態を汚染しないことを確認する
2. **各ページで異なる値を入力する**：同じ値では「前の入力が残っているか、たまたま同じ値か」を区別できない
3. **`waitFor` で非同期的な DOM 更新を待つ**：Jotai の atom 更新はバッチ処理されることがあるため、同期的なアサーションでは信頼性が低い

---

## 調査のまとめと得られた知識

### バグ調査の結論

サインアップフォームの値がページ遷移後も残るバグは **コミット `ca13d16` で修正済み** であり、現在のコードベースでは再現しない。

調査によって確認できたこと：
- フォーム状態は `useAuthForm` 内の `useState` のみで管理されており、Jotai atom への漏洩はない
- `SignupPage` と `LoginPage` は異なるコンポーネント型であり、型変化だけでアンマウントが保証される
- `key` プロップは将来的な型統合リファクタリングへの防御として機能している
- 回帰テスト（signup → login → signup の往復）が存在し、全 180 件のテストがパス済み

### key プロップの正しい理解

`key` プロップの役割は「状態をリセットする」ことではなく、「このコンポーネントインスタンスの同一性を React に伝える」ことだ。`key` が変われば React は前のインスタンスを捨てて新しく作るため、結果として `useState` がリセットされる。

- 型が変わるだけでもリセットは起きる（`key` は不要だが、あると将来の変更に対して安全）
- 型が同じなら `key` がなければリセットは起きない（`key` が必須）
- `key` に不安定な値（`Math.random()`, 配列インデックスなど）を使うと意図しない再マウントが起きる

### ナビゲーション起因の状態バグを防ぐチェックリスト

```
☑ フォーム状態は useState または useReducer のローカル管理か？
  （グローバル atom に入れていないか）

☑ 条件付きレンダリングで同一型のコンポーネントを切り替えている場合、key が付いているか？

☑ 将来のリファクタリングで型統合が起きても安全なように key が付いているか？

☑ ナビゲーションの「往復」を含む回帰テストが存在するか？
```

---

## 参考

- [React 公式ドキュメント — Preserving and Resetting State](https://react.dev/learn/preserving-and-resetting-state)
- [React 公式ドキュメント — Rendering Lists: key](https://react.dev/learn/rendering-lists#keeping-list-items-in-order-with-key)
