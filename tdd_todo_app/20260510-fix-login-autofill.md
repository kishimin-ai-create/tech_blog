# ログイン画面で保存済み認証情報が自動入力されるバグの修正（autoComplete 属性の誤設定）

## エラー概要

ログイン画面（`LoginPage`）を表示すると、メールアドレスとパスワードのフィールドが**ユーザーが何も入力していないのに最初から埋まった状態**で表示されるという不具合が発生していた。

React の `useState('')` でフィールドを空文字列に初期化しているにもかかわらず、Chrome のクレデンシャルマネージャーが保存済みの認証情報をフィールドに流し込んでいたことが原因である。

---

## 原因

### `autoComplete` 属性がブラウザを「招待」していた

修正前の `LoginPage.tsx` には次のように記述されていた。

```tsx
// 修正前
<input
  id="email"
  type="email"
  value={email}
  onChange={(e) => setEmail(e.target.value)}
  autoComplete="username"          // ← メールフィールド
/>

<input
  id="password"
  type="password"
  value={password}
  onChange={(e) => setPassword(e.target.value)}
  autoComplete="current-password"  // ← パスワードフィールド
/>
```

`autoComplete="username"` と `autoComplete="current-password"` はどちらも **HTML の仕様で定義された正式な値**であり、ブラウザに対して「このフィールドにはユーザー名・現在のパスワードを自動入力してよい」と明示的に許可するシグナルとなる。Chrome はこのシグナルを受け取り、保存済みの認証情報をページロード後に自動的に流し込む。

### `useState('')` だけでは不十分な理由

```tsx
const [email, setEmail] = useState('')
const [password, setPassword] = useState('')
```

`useState('')` は React の初期レンダリング時に空文字列をセットする。しかしブラウザの自動入力は **React のマウント完了後に非同期で発火**する。これは React の制御外で起きる DOM 操作であるため、React の状態管理だけでは防ぐことができない。

結果として次のような流れが生じていた。

```
1. React がコンポーネントをマウント → email='', password='' でレンダリング
2. ブラウザが autoComplete 属性を認識 → 保存済み認証情報を DOM に書き込む
3. 画面には保存済みの値が表示される（React の state は '' のまま）
```

### 経緯：SignupPage への対応が起点だった

コミット `bc16cad` では、SignupPage 側の自動入力バグ（フォームページ遷移時に前回の値が残る）を修正するために `autoComplete` 属性を追加した。その際、SignupPage のメールには `autoComplete="off"` を設定した一方、LoginPage には `autoComplete="username"` と `autoComplete="current-password"` を設定した。

これはログインフォームにとって **HTML 仕様として意味的には正しい選択**だった。しかし今回のアプリの UX 要件は「ページ遷移のたびにフィールドを空にする」というものであり、ブラウザが積極的に認証情報を補完する動作は要件と相反していた。結果として `bc16cad` は LoginPage に新たな自動入力バグを作り込むことになった。

---

## 修正内容

### 対象ファイル

- `frontend/src/features/auth/pages/LoginPage.tsx`
- `frontend/src/features/auth/pages/LoginPage.test.tsx`

### コード修正（LoginPage.tsx）

両フィールドの `autoComplete` を `"off"` に変更した。これは SignupPage のメールフィールドにすでに適用されていたアプローチと同じである。

```tsx
// 修正後
<input
  id="email"
  type="email"
  value={email}
  onChange={(e) => setEmail(e.target.value)}
  autoComplete="off"   // "username" から変更
/>

<input
  id="password"
  type="password"
  value={password}
  onChange={(e) => setPassword(e.target.value)}
  autoComplete="off"   // "current-password" から変更
/>
```

`autoComplete="off"` はブラウザに対して「このフィールドへの自動入力を抑制してほしい」と伝える。Chrome が `off` を完全に無視するケースも過去には存在したが、`username` / `current-password` という明示的な招待を取り除くだけで実用上の自動入力は止まる。

### `autoComplete` 各値の比較

| 値 | 意味 | ブラウザの挙動 |
|---|---|---|
| `"username"` | ユーザー名（メールアドレス含む）フィールド | 保存済み認証情報を積極的に補完する |
| `"current-password"` | 現在のパスワードフィールド | 保存済みパスワードを積極的に補完する |
| `"new-password"` | 新規パスワード登録フィールド | 保存済みパスワードの補完を抑制し、強力なパスワードを提案する |
| `"off"` | 自動入力を明示的に抑制 | フィールドへの自動補完を行わない |

---

## TDD アプローチ：RED → GREEN

今回の修正は TDD の手順に従って進められた。

### ステップ 1：失敗するテストを先に書く（RED）

`LoginPage.test.tsx` に以下の 2 つのテストを追加した。

```tsx
it('when rendered, then email input has autoComplete set to off to prevent browser autofill', () => {
  // Arrange + Act
  renderWithProviders(<LoginPage />)

  // Assert
  const emailInput = screen.getByRole('textbox', { name: /email/i })
  expect(emailInput).toHaveAttribute('autocomplete', 'off')
})

it('when rendered, then password input has autoComplete set to off to prevent browser autofill', () => {
  // Arrange + Act
  renderWithProviders(<LoginPage />)

  // Assert
  const passwordInput = screen.getByLabelText(/password/i)
  expect(passwordInput).toHaveAttribute('autocomplete', 'off')
})
```

テスト追加時点では `LoginPage.tsx` がまだ `autoComplete="username"` と `autoComplete="current-password"` を持っていたため、これらのテストは意図通りに失敗した（RED）。

### ステップ 2：実装を修正してテストを通す（GREEN）

`LoginPage.tsx` の両フィールドを `autoComplete="off"` に変更したところ、追加した 2 つのテストを含む全テストがパスした（GREEN）。

テストが検証する内容は「`autocomplete` 属性の値が `"off"` であること」のみであるため、将来また誤った値に戻れば即座にテストが失敗する。意図しない回帰を防ぐ安全網として機能する。

---

## ポイント・注意点

**意味的な正しさと UX 要件は必ずしも一致しない**

`autoComplete="username"` と `autoComplete="current-password"` はログインフォームに対して HTML 仕様上は適切な値である。しかしこのアプリでは「ページ遷移のたびにフォームを空にする」という要件が優先され、ブラウザによる積極的な補完は望ましくなかった。仕様として正しい属性値がアプリの動作要件と衝突することがある点に注意が必要である。

**ブラウザ自動入力は React の状態管理の外側で起きる**

React の `useState` や `defaultValue` はサーバーサイドまたは初回レンダリング時の初期値を制御するものであり、マウント後にブラウザが DOM を直接書き換える自動入力には干渉できない。この領域の制御は HTML 属性（`autoComplete`）で行う必要がある。

---

## まとめ

| 項目 | 内容 |
|---|---|
| **バグ** | ログイン画面の初期表示時にフィールドが保存済み認証情報で埋まる |
| **根本原因** | `autoComplete="username"` / `"current-password"` がブラウザの自動入力を招待していた |
| **修正** | 両フィールドを `autoComplete="off"` に変更 |
| **変更ファイル** | `LoginPage.tsx`、`LoginPage.test.tsx` |
| **TDD** | 失敗テスト 2 本を先に追加（RED）→ 実装修正でパス（GREEN） |
| **前提コミット** | `bc16cad` で SignupPage に適用済みのアプローチを LoginPage にも適用 |

`autoComplete` 属性は一見細かいディテールだが、ブラウザの自動入力動作を直接制御する重要な属性である。フォームを持つ画面を実装・レビューする際には、各フィールドの `autoComplete` 値がアプリの UX 要件と一致しているかを確認することを習慣化しておきたい。
