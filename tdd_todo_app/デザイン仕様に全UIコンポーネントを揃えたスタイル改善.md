# デザイン仕様に全UIコンポーネントを揃えたスタイル改善

## 対象読者

- React + Tailwind CSS でフロントエンド開発をしている方
- デザイン仕様書とコードの乖離に悩んでいる方
- アクセシビリティ（WCAG AA）対応をこれから取り組む方

---

## この記事の範囲

**カバーする内容**

- コミット `fa7011a`（`style: align all UI components with design spec`）で変更された 9 ファイルの内容
- 各コンポーネントの変更前後の比較
- アクセシビリティ対応（`focus-visible`）の実装方法
- UI 変更に追随して必要になったテスト修正

**カバーしない内容**

- ビジネスロジックの変更（本コミットはスタイルのみ）
- Tailwind CSS の設定・カスタムテーマ
- デザインシステムの設計経緯

---

## 背景：デザイン仕様との乖離

`docs/design/frontend/ui.md` にはレイアウトとスタイルの要件が定義されています。たとえば App Detail ページのヘッダーは以下のレイアウトを求めています。

```
┌─────────────────────────────────────┐
│  [← Back]  App Name  [Edit] [Delete]│ (Header)
├─────────────────────────────────────┤
│  Created: 2026-04-12                │
│  Updated: 2026-04-12                │
```

しかし変更前の実装では、`← Back` ボタンが独立した縦方向ブロックに配置されており、`Updated` の日付も表示されていませんでした。また、ボタンにはホバー状態こそあれ、キーボード操作時のフォーカスリングが定義されておらず、アクセシビリティ要件を満たしていない状態でした。

このコミットはそれらの乖離をすべて解消するための**スタイル専用の修正**です。

---

## 変更内容

### 1. AppHeader.tsx — ヘッダーレイアウトの再構築

最も変更量が多かったのが `AppHeader.tsx` です。

**変更前**

```tsx
<div className="flex justify-between items-center mb-6">
  <div>
    <button className="mb-2 text-sm text-blue-500 hover:underline">
      Back
    </button>
    <h1 className="text-2xl font-bold">{app.name}</h1>
    <p className="text-sm text-gray-500">Created: {app.createdAt.slice(0, 10)}</p>
  </div>
  <div className="flex gap-2">
    <button ...>Edit</button>
    <button ...>Delete</button>
  </div>
</div>
```

**変更後**

```tsx
<div className="mb-6">
  <div className="flex items-center justify-between">
    <div className="flex items-center gap-3">
      <button
        className="rounded text-sm text-blue-500 hover:underline
                   focus-visible:outline focus-visible:outline-2 focus-visible:outline-blue-500"
      >
        ← Back
      </button>
      <h1 className="text-2xl font-bold">{app.name}</h1>
    </div>
    <div className="flex gap-2">
      <button className="... focus-visible:outline focus-visible:outline-2 focus-visible:outline-gray-400">
        Edit
      </button>
      <button className="... focus-visible:outline focus-visible:outline-2 focus-visible:outline-red-500">
        Delete
      </button>
    </div>
  </div>
  <div className="mt-2 space-y-0.5 text-sm text-gray-500">
    <p>Created: {app.createdAt.slice(0, 10)}</p>
    <p>Updated: {app.updatedAt.slice(0, 10)}</p>
  </div>
</div>
```

**ポイント**

- `← Back`・App Name・`Edit`・`Delete` が 1 行に横並びになり、仕様通りのレイアウトに
- `← ` プレフィックスを Back ボタンに追加し、ナビゲーション方向が視覚的に明確に
- `Updated: YYYY-MM-DD` の表示を新たに追加
- 全 3 ボタンに `focus-visible:outline` を付与

---

### 2. TodoItem.tsx — カードのビジュアル強化と作成日表示

**変更前**

```tsx
<div className="p-3 border rounded bg-white">
```

**変更後**

```tsx
<div className="p-4 border border-gray-200 rounded-lg bg-white shadow-sm">
```

また、Todo タイトルの下に作成日を追加しました。

```tsx
<p className="text-xs text-gray-400 ml-6 mt-0.5">
  Created: {todo.createdAt.slice(0, 10)}
</p>
```

仕様書の TodoItem レイアウトには `Created: 2026-04-12` が含まれており、これを実装しました。`ml-6` はチェックボックスの幅に合わせたインデントです。

---

### 3. AppCard.tsx — ホバー効果とフォーカスリング

**変更前後の差分（主要部分）**

| 要素 | 変更前 | 変更後 |
|------|--------|--------|
| カードコンテナ | `rounded shadow hover:shadow-lg` | `rounded-lg shadow-sm hover:shadow-md transition-shadow duration-200` |
| アプリ名 | （色指定なし） | `text-gray-900` |
| 日付 | マージンなし | `mt-1` |
| View ボタン | フォーカスリングなし | `focus-visible:outline focus-visible:outline-2 focus-visible:outline-blue-500 transition-colors duration-150` |

ホバー時の影は `shadow-lg`（強め）から `shadow-md`（控えめ）に下げ、`transition-shadow` でアニメーションを付けることで動きが自然になりました。

---

### 4. AppForm.tsx — 送信ボタンの disabled 状態改善

```tsx
// 変更前
className="... disabled:opacity-50"

// 変更後
className="... disabled:opacity-50 disabled:cursor-not-allowed
           focus-visible:outline focus-visible:outline-2 focus-visible:outline-blue-500
           transition-colors duration-150"
```

ローディング中（`isLoading === true`）に `cursor-not-allowed` が付くことで、ユーザーがボタンが無効であることを視覚とカーソル変化の両方で確認できます。

---

### 5. AppDetailPage.tsx — 削除確認ダイアログの改善

削除確認ダイアログはメッセージとビジュアルの両方を改善しました。

**変更前**

```
Delete this app and all its todos?
[Confirm] [Cancel]
```

**変更後**

```
⚠️ Warning: This action cannot be undone
Deleting this app will permanently remove it and all its associated todos.
[Confirm] [Cancel]
```

`⚠️` の絵文字と `font-semibold text-yellow-800` による強調で、操作の不可逆性が伝わりやすくなりました。ダイアログ内の 2 ボタンにも `focus-visible:outline` を追加しています。

---

### 6. TodoForm.tsx — 入力フォームのビジュアル整備

フォームコンテナとタイトル入力フィールドのスタイルを整えました。

```tsx
// フォームコンテナ
<form className="p-4 border border-gray-200 rounded-lg bg-gray-50">

// タイトル入力
<input className="w-full px-3 py-2 border border-gray-300 rounded text-sm
                  focus:outline-none focus:ring-2 focus:ring-blue-500" />
```

タイトル入力欄には `focus:ring-2 focus:ring-blue-500` を追加しました。これは仕様書のスタイリングガイドラインに記載されているサンプルコードに準拠した形です。

---

### 7. AppListPage.tsx — Create App ボタン

```tsx
// 変更後
className="px-4 py-2 bg-blue-500 text-white rounded hover:bg-blue-600
           focus-visible:outline focus-visible:outline-2 focus-visible:outline-blue-500
           transition-colors duration-150"
```

ページ内で唯一のアクションボタンにも統一的なフォーカスリングを追加しました。

---

## アクセシビリティ対応：`focus-visible` の使い方

今回の変更で最も横断的に適用されたのが `focus-visible:outline` です。

### `focus` と `focus-visible` の違い

| クラス | 発火タイミング |
|--------|--------------|
| `focus:` | クリックを含む全フォーカス |
| `focus-visible:` | キーボード操作など、ブラウザが「フォーカスリングが必要」と判断したとき |

マウスクリックでボタンを押したときにリングが表示されるのは視覚的に冗長なため、`focus-visible:` を使うのが現代的なベストプラクティスです。WCAG 2.1 AA の達成基準 2.4.7（フォーカスの可視化）を満たすためにも、キーボードユーザーへのフォーカスインジケーター提供は必須です。

### 本プロジェクトでの統一パターン

全ボタンで以下の 3 クラスをセットで使用することにしました。

```
focus-visible:outline
focus-visible:outline-2
focus-visible:outline-{color}
```

`{color}` はボタンの種類に応じて使い分けています。

| ボタン種別 | フォーカスリング色 |
|-----------|-----------------|
| プライマリ（青） | `focus-visible:outline-blue-500` |
| セカンダリ（グレー） | `focus-visible:outline-gray-400` |
| 危険操作（赤） | `focus-visible:outline-red-500` |

---

## テストへの影響

UI 変更によってテストが 2 件壊れました。どちらも同じ原因です。

### 原因

`AppHeader` に `Updated` 日付を追加したことで、同じ日付文字列（`2026-04-01`）が画面内に複数存在するようになりました。`getByText` は要素が 1 つだけ存在することを前提としているため、`Multiple elements found` エラーが発生します。

### 修正方針

```tsx
// 変更前（1 要素を想定）
expect(screen.getByText(/2026-04-01/)).toBeInTheDocument()

// 変更後（複数要素を許容）
expect(screen.getAllByText(/2026-04-01/).length).toBeGreaterThan(0)
```

`findAllByText` の非同期版も同様に変更しました（`AppDetailPage.test.tsx`）。

この修正は「日付が画面に表示されている」という意図を正確に保ちながら、複数要素の存在も許容する形になっています。

---

## まとめ

| 観点 | 変更内容 |
|------|---------|
| **レイアウト** | AppHeader を仕様通りの 1 行横並びレイアウトに再構築 |
| **情報表示** | `Updated` 日付（AppHeader）・`Created` 日付（TodoItem）を追加 |
| **カードデザイン** | `rounded-lg`・`shadow-sm`・`border-gray-200` で統一感を向上 |
| **インタラクション** | ホバー遷移に `transition-colors/shadow` を追加 |
| **アクセシビリティ** | 全ボタンに `focus-visible:outline` を付与（WCAG AA 準拠） |
| **UX 改善** | 削除確認に警告文と `⚠️` を追加、disabled 時に `cursor-not-allowed` |
| **テスト** | 複数日付要素の存在を許容するよう `getAllByText` に変更 |

今回の変更はすべてスタイル層のみで、ロジックには手を触れていません。デザイン仕様書（`docs/design/frontend/ui.md`）を単一の真実として参照し、コードとの乖離を一括で解消する作業でした。「仕様書とコードを常に同期させる」という原則をデザイン面でも維持する取り組みとして位置づけられます。
