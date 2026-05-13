# React の key プロップが無いとフォーム値が保持される問題を修正した

## 対象読者

- React を使ったフロントエンド開発者
- useState フックの動作を理解したい開発者
- コンポーネントのライフサイクルに興味がある開発者

## 記事が扱う範囲

このサムマリーは、SignupPage と LoginPage 間でのフォーム値の保持バグと、その修正方法に限定されています。他の認証フローやページナビゲーションの仕様については扱いません。

## 問題の背景

ユーザーが SignupPage でメールアドレスとパスワードを入力した後、LoginPage に切り替えてから SignupPage に戻ると、以前入力した値がそのまま表示されたままになっていました。これは予期しない振る舞いです。通常、ページを切り替えるたびにフォームはリセットされるべきです。

## 原因：key プロップの欠落

React では、条件付きレンダリングされるコンポーネントの`key`プロップを省略すると、以下の問題が発生します。

### React がコンポーネントのアイデンティティを判定できない

App.tsx の変更前のコード（推測）：

```javascript
if (currentPage.name === 'signup') return <SignupPage />
if (currentPage.name === 'login') return <LoginPage />
```

この場合、React は以下のように判定してしまいます：

1. ページ A で `<SignupPage />` をレンダリング
2. ページ B で `<LoginPage />` をレンダリング（条件分岐が異なるため）
3. しかし React は**同じポジションの別の JSX 要素**として判定する傾向があります

### useState フックが新しい状態を初期化しない

SignupPage と LoginPage が異なるコンポーネントでも、React が同じ「スロット」として扱うと、以下が起きます：

- SignupPage で `useState` で保持されていた入力値（例：`signupEmail = 'signup@example.com'`）
- ページを切り替えても、React はそのコンポーネントを**完全にアンマウント**しません
- LoginPage がマウントされるとき、SignupPage の state が裏側で残ったままになり、ユーザーが戻ったとき復元される

これは React の一般的な落とし穴で、`key`プロップなしの条件付きレンダリングで特に起きやすいです。

## 解決策：key プロップを追加する

変更後の App.tsx：

```javascript
if (currentPage.name === 'signup') return <SignupPage key="signup" />
if (currentPage.name === 'login') return <LoginPage key="login" />
```

### key プロップが何をするか

- **一意な識別子を付与** — `key="signup"` と `key="login"` により、React は 2 つのコンポーネントが完全に異なるインスタンスであることを明確に認識します
- **確実なマウント/アンマウント** — page 名が切り替わると、前のコンポーネントは完全にアンマウントされ、新しいコンポーネントが新しい state で初期化されてマウントされます
- **state の独立性を保証** — 各ページのフォーム値は独立して管理され、ページ切り替え時に相互に干渉しなくなります

## 実装の詳細

### 変更ファイル

**frontend/src/App.tsx**
- Line 21: `<SignupPage key="signup" />` - key を追加
- Line 22: `<LoginPage key="login" />` - key を追加

### テストで検証

新しいテスト（frontend/src/App.test.tsx, Lines 66-116）で以下を検証：

1. SignupPage にメール・パスワードを入力
2. LoginPage に切り替え → フォーム値が**空（リセット）される**ことを確認
3. LoginPage にメール・パスワードを入力
4. SignupPage に戻す → フォーム値が**空（リセット）される**ことを確認

このテストにより、フォーム値が独立していることが保証されます。

## key プロップを使うときの注意点

### ✅ 推奨される使い方

```javascript
// コンポーネントの identity が異なるとき
return currentPage.name === 'signup' ? (
  <SignupPage key="signup" />
) : (
  <LoginPage key="login" />
)
```

```javascript
// 条件分岐でコンポーネントが入れ替わるとき
if (mode === 'edit') return <EditForm key="edit" />
if (mode === 'view') return <ViewForm key="view" />
```

### ❌ してはいけないこと

```javascript
// key に index を使わない（配列が並べ替わると壊れる）
items.map((item, index) => <Item key={index} />)

// key に random 値を使わない（毎回新しいコンポーネントになる）
<MyComponent key={Math.random()} />
```

## まとめ

条件付きレンダリングされるコンポーネントに `key` プロップを追加することで：

1. React がコンポーネントを正しく識別できるようになる
2. ページ切り替え時にコンポーネントが確実にアンマウント/マウントされる
3. 各ページのフォーム state が独立して保たれる

このパターンは、認証フロー（Login/Signup 切り替え）、複数フォーム、モーダルの表示/非表示など、状態を持つコンポーネントを条件分岐でレンダリングするあらゆる場面で活用できます。

## 関連リソース

- [React 公式ドキュメント - Rendering Lists](https://react.dev/learn/rendering-lists)
- [React 公式ドキュメント - Keys](https://react.dev/learn/rendering-lists#keeping-list-items-in-order-with-key)
- [State as a Snapshot - React ドキュメント](https://react.dev/learn/state-as-a-snapshot)
