# Todo App にスティッキーナビバーを追加した — Tailwind CSS だけで作るヘッダー改善

**対象読者:** React と Tailwind CSS を使うフロントエンドエンジニア。ヘッダーのレイアウト実装やアクセシビリティ対応に興味がある方。  
**スコープ:** `frontend/src/App.tsx` の変更のみを扱います。バックエンド・テストコードは対象外です。

---

## 背景

ログイン後に表示されるメイン画面（`App.tsx` の認証済みブランチ）は、以前は次のような最小限の構造だった。

```tsx
// 変更前
return (
  <div>
    <LogoutButton />
    <button type="button" onClick={goToUserProfile}>
      プロフィール
    </button>
    <AppListPage />
    {/* ... */}
  </div>
)
```

スタイルは一切なく、ボタンは左上にただ並んでいた。スクロールしてもヘッダーは固定されず、コンテンツ幅も制御されていない状態だった。

このコミットでは次の 3 点を解決した。

1. アプリタイトル・操作ボタンをまとめた**スティッキーヘッダー**を追加する
2. コンテンツ領域を**最大幅と余白で整える**
3. プロフィールボタンに**視覚的な識別性とアクセシビリティ属性**を与える

---

## 実装の全体構造

変更後の `App.tsx` のレイアウト骨格は次のとおり。

```tsx
return (
  <div className="min-h-screen bg-gray-50">
    {/* ① スティッキーヘッダー */}
    <header className="sticky top-0 z-50 w-full border-b border-gray-200 bg-white shadow-sm">
      <div className="mx-auto flex h-14 max-w-5xl items-center justify-between px-4 sm:px-6">
        <span className="text-base font-semibold tracking-tight text-gray-800">
          Todo App
        </span>
        <div className="flex items-center gap-2">
          <button /* プロフィールボタン */ />
          <LogoutButton />
        </div>
      </div>
    </header>

    {/* ② メインコンテンツ */}
    <main className="mx-auto max-w-5xl px-4 py-6 sm:px-6">
      <AppListPage />
      {/* ... */}
    </main>
  </div>
)
```

アウターラッパーの `min-h-screen bg-gray-50` がページ全体の背景色を設定し、ヘッダーとコンテンツの両方が同じ `max-w-5xl` 制約に収まるため、横方向の整合性が保たれる。

---

## ① スティッキーヘッダーの実装

### `sticky top-0 z-50` の組み合わせ

```tsx
<header className="sticky top-0 z-50 w-full border-b border-gray-200 bg-white shadow-sm">
```

| クラス | 役割 |
|---|---|
| `sticky top-0` | スクロール時もビューポート上端に固定 |
| `z-50` | モーダルやドロップダウンより下だが通常コンテンツの上に重なる |
| `w-full` | ビューポート幅いっぱいに広がる |
| `bg-white` | ヘッダー背景を白で塗りつぶし、コンテンツが透けて見えないようにする |
| `border-b border-gray-200` | 下線でコンテンツとの境界を示す |
| `shadow-sm` | 控えめな影で奥行き感を演出 |

`sticky` は `fixed` と異なり通常のドキュメントフローに残るため、後続の `<main>` がヘッダーの高さ分を自動的に下にずれる。高さは `h-14`（56px）で固定しており、コンテンツが少なくても上下に均等な余白が生まれる。

### タイトルとボタンの配置

内側のコンテナは `flex items-center justify-between` で左右に分割している。

```tsx
<div className="mx-auto flex h-14 max-w-5xl items-center justify-between px-4 sm:px-6">
  {/* 左: アプリ名 */}
  <span className="text-base font-semibold tracking-tight text-gray-800">
    Todo App
  </span>

  {/* 右: ボタン群 */}
  <div className="flex items-center gap-2">
    {/* プロフィール / ログアウト */}
  </div>
</div>
```

`max-w-5xl` をヘッダーとコンテンツの両方に共通指定することで、幅の広い画面でも左端・右端が揃う。`px-4 sm:px-6` はモバイルファーストのレスポンシブ余白で、小さい画面では 16px、sm ブレークポイント以上では 24px のパディングが付く。

---

## ② プロフィールボタンの視覚とアクセシビリティ

### ブルー系のカラーリング

プロフィールボタンは `LogoutButton`（グレー系）と視覚的に区別するため、ブルー系のカラーをつけた。

```tsx
className="flex items-center gap-1.5 rounded bg-blue-50 px-4 py-2 text-sm font-medium text-blue-700
           hover:bg-blue-100 focus-visible:outline focus-visible:outline-2 focus-visible:outline-blue-500
           active:bg-blue-200 transition-colors duration-150"
```

| 状態 | 背景色 | 文字色 |
|---|---|---|
| 通常 | `bg-blue-50` | `text-blue-700` |
| ホバー | `bg-blue-100` | 変化なし |
| アクティブ（押下） | `bg-blue-200` | 変化なし |
| フォーカス | アウトライン 2px、`outline-blue-500` | 変化なし |

`focus-visible:outline` でフォーカスリングを視覚化する。`focus-visible` を使うことで、マウス操作時には余分なリングが出ず、キーボードナビゲーション時にのみ表示される。

`px-4 py-2 text-sm font-medium rounded` は `LogoutButton` と同じボタンサイズを想定した指定で、2つのボタンが横に並んだときに高さが揃う。

### インライン SVG アイコン

ユーザーアイコンは外部ライブラリを使わずインライン SVG で実装した。

```tsx
<svg
  xmlns="http://www.w3.org/2000/svg"
  className="h-4 w-4 shrink-0"
  viewBox="0 0 24 24"
  fill="currentColor"
  aria-hidden="true"
>
  <path d="M12 12c2.7 0 4.8-2.1 4.8-4.8S14.7 2.4 12 2.4 7.2 4.5 7.2 7.2 9.3 12 12 12zm0 2.4c-3.2 0-9.6 1.6-9.6 4.8v2.4h19.2v-2.4c0-3.2-6.4-4.8-9.6-4.8z" />
</svg>
```

`aria-hidden="true"` でスクリーンリーダーがアイコンを読み飛ばすよう指示している。ボタン自体に `aria-label="プロフィールページへ移動"` が付いているため、アイコンを二重に読み上げる必要がない。

```tsx
<button
  type="button"
  onClick={goToUserProfile}
  aria-label="プロフィールページへ移動"
  className="..."
>
  <svg aria-hidden="true">...</svg>
  プロフィール
</button>
```

**ここでの原則は「No ARIA is better than bad ARIA」** — アイコンに説明的な `aria-label` を付けるより、ボタン側で一つの意味のある名前を持たせる方がシンプルかつ確実である。`fill="currentColor"` により、文字色（`text-blue-700`）と同じ色がアイコンに自動適用される。

`shrink-0` はフレックスアイテムとしてアイコンが圧縮されるのを防ぐ。ラベルテキストが長くなってもアイコンが潰れない。

---

## ③ コンテンツ領域の整備

```tsx
<main className="mx-auto max-w-5xl px-4 py-6 sm:px-6">
  <AppListPage />
  {/* ... */}
</main>
```

変更前はコンテンツが `<div>` 直下に並んでいて幅の制御がなかった。`<main>` に変更したことで HTML のセマンティクスが改善され、スクリーンリーダーがページの主要コンテンツ領域を正しく識別できる。

`py-6` はヘッダー直下の上余白として 24px を確保し、コンテンツが詰まって見えるのを防ぐ。

---

## 新しい npm 依存は追加しない方針

アイコンの実装にあたって `lucide-react` や `@heroicons/react` などのアイコンライブラリは導入しなかった。インライン SVG で完結することで、バンドルサイズを増やさずに目的を達成している。

アイコンを 1〜2 個追加するためだけにライブラリを導入するのは YAGNI（You Aren't Gonna Need It）の観点でも避けるべきトレードオフである。

---

## まとめ

| 変更点 | 対処した問題 |
|---|---|
| `sticky top-0 z-50` ヘッダー | スクロール時にナビゲーションが消える |
| `max-w-5xl` + `justify-between` | 横幅が広い画面でのレイアウト崩れ |
| ブルー系プロフィールボタン | ログアウトボタンとの視覚的区別がない |
| `aria-label` + `aria-hidden="true"` | キーボード・スクリーンリーダーでの操作性 |
| `<main>` ラッパー | コンテンツ領域のセマンティクス欠如 |
| インライン SVG | npm 依存を増やさずにアイコンを追加 |

Tailwind CSS の `sticky`、`z-50`、`focus-visible` といったユーティリティクラスを組み合わせるだけで、新たなライブラリを加えることなく機能的で保守しやすいヘッダーを実装できる。アクセシビリティについても、`aria-label` を正しい要素に付け `aria-hidden` で装飾要素を隠すという最小限の対応で、スクリーンリーダー体験を損なわない構造になっている。
