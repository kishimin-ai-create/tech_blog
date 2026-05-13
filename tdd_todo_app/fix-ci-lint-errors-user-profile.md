# UserProfilePage と関連テストの CI Lint エラー修正

## 概要

ユーザープロファイル編集機能を実装した後、CI パイプラインがフロントエンドコンポーネント、バックエンドテストファイル、GitHub Actions ワークフロー全体で4種類の lint 違反を検出しました。この記事では各エラー、その根本原因、そして適用した具体的な修正を説明します——同じパターンを将来の作業で避けるために。

**影響を受けたファイル：**

| ファイル | 違反したルール |
|---|---|
| `frontend/src/features/auth/pages/UserProfilePage.tsx` | `import/order`、`no-misused-promises` |
| `backend/src/infrastructure/user-profile-validation.small.test.ts` | `vitest/no-conditional-expect`、`stylistic/semi` |
| `backend/src/tests/user-profile.medium.test.ts` | `stylistic/semi` |
| `.github/workflows/ci-nightly.yml` | 前のステップが失敗するとステップがスキップされる |

---

## 修正1 — `import/order`：UserProfilePage.tsx の間違ったインポート順序

### エラー

```
import/order: 'jotai' import must come before 'react'
```

### 原因

プロジェクトの ESLint 設定（`eslint-plugin-import` 経由）は、外部パッケージのインポートをアルファベット順にソートすることを強制します。元のコードは `jotai` より前に `react` をリストしていました：

```ts
// ❌ 修正前
import { useState } from 'react'
import { useAtom } from 'jotai'
```

`j` は `r` より前に来るため、`jotai` が先に現れる必要があります。

### 修正

2つの行を入れ替えます：

```ts
// ✅ 修正後
import { useAtom } from 'jotai'
import { useState } from 'react'
```

実行時の動作は変わりません——これはインポートをスキャンしやすく、コードベース全体で一貫性を保つための純粋に外観上のルールです。

---

## 修正2 — `no-misused-promises`：async 関数が直接 `onSubmit` に渡されている

### エラー

```
no-misused-promises: Promise-returning function provided to attribute where a void return was expected
```

### 原因

`handleSubmit` は `async` 関数であり、`Promise` を返します。React の `onSubmit` prop は `void` を返すハンドラーを期待しています。async 関数を直接代入するとこのコントラクトに違反します：

```tsx
// ❌ 修正前 — handleSubmit は Promise<void> を返し、void ではない
<form onSubmit={handleSubmit}>
```

`Promise` が予期せず reject した場合、React によってリジェクションがサイレントに飲み込まれます。TypeScript/ESLint ルール `@typescript-eslint/no-misused-promises` はまさにこのクラスのバグを表面化させるために設計されています。

### 修正

async 呼び出しを同期ラムダでラップし、`.catch()` をチェーンしてエラーハンドリングを明示します：

```tsx
// ✅ 修正後
<form onSubmit={(e) => { handleSubmit(e).catch(() => { /* handled inside */ }) }}>
```

コメント `/* handled inside */` は、エラーハンドリングがすでに `handleSubmit` 自体の中（`setError` を呼び出す `try/catch` ブロック）にあることを示しています。`.catch()` ラッパーは `void` 戻り値の要件を満たし、未処理の rejection 警告を防ぐためだけに存在しています。

---

## 修正3 — `vitest/no-conditional-expect`：テスト内の `expect()` を囲む条件分岐

### エラー

```
vitest/no-conditional-expect: Expect must not be called in a conditional
```

### 原因

`user-profile-validation.small.test.ts` のいくつかのテストが、判別ユニオンメンバーにアクセスする前に `if` ガードを使用していました：

```ts
// ❌ 修正前 — expect() が if ブロックの中にある
const result = parseUpdateUserProfileInput(body)
if (result.success) {
  expect(result.email).toBe('valid@example.com')
}
```

この問題は2つあります：

1. **ルール違反**：`vitest/no-conditional-expect` は条件分岐の中の `expect()` 呼び出しを禁止します。条件が `false` の場合、アサーションはテストを失敗させるのではなくサイレントにスキップされるからです。`result.success` が `false` の場合でもテストはパスしてしまいます。
2. **TypeScript の絞り込みが失われる**：`if` ブロックの後、`result` はもはや成功ブランチに絞り込まれていないため、`result.email` のようなプロパティは外では型安全ではありません。

このパターンは `email`、`currentPassword`、`newPassword` フィールドのアサーションをカバーするファイル全体で6回現れていました。

### 修正

Vitest から `assert` をインポートし、`expect()` の前に無条件に呼び出します：

```ts
// ✅ 修正後 — assert() は false の場合スローするため、expect() が常に到達する
import { assert, describe, expect, it } from 'vitest'

const result = parseUpdateUserProfileInput(body)
assert(result.success)               // false の場合スロー（テスト失敗）
expect(result.email).toBe('valid@example.com')  // 型安全で常に実行される
```

`assert(result.success)` は型ガードとして機能します：パスすると、TypeScript はその後のすべての行で `result` が成功ブランチにあることを認識します。失敗すると、テストはサイレントにパスする代わりに明確なメッセージで即座に失敗します。

`if (result.success) { expect(...) }` パターンの6つのすべての出現がこの2行の形式に置き換えられました。

---

## 修正4 — `stylistic/semi`：バックエンドテストファイルのセミコロン不足

### エラー

```
stylistic/semi: Missing semicolon
```

### 原因

バックエンドの ESLint 設定はトレイリングセミコロンを強制します（`stylistic/semi`）。`user-profile-validation.small.test.ts` と `user-profile.medium.test.ts` の両方が文末セミコロンなしで書かれていましたが、これは Red（失敗テスト）と Green（実装）フェーズで使われたスタイルと一致しつつも、強制されたルールとは異なっていました。

### 修正

ESLint の自動修正を実行します：

```bash
npx eslint --fix backend/src/infrastructure/user-profile-validation.small.test.ts
npx eslint --fix backend/src/tests/user-profile.medium.test.ts
```

これにより両方のファイルのすべての文に一発でセミコロンが追加されました。ロジックの変更はありません。

---

## 修正5 — CI ナイトリー：前のステップが失敗したときに大規模 E2E ステップがスキップされる

### 問題

`.github/workflows/ci-nightly.yml` で、「Run large E2E tests」ステップに `if:` 条件がありませんでした。GitHub Actions は前のステップが失敗すると後続のステップをスキップするため、「Run medium E2E tests」ステップが失敗すると、大規模 E2E テストはまったく実行されず——そのスイートでの潜在的な失敗が隠されていました。

```yaml
# ❌ 修正前 — medium E2E ステップが失敗するとスキップされる
- name: Run large E2E tests
  run: npm run test:e2e:large
```

### 修正

前のステップの結果に関わらずステップを強制的に実行するために `if: always()` を追加します：

```yaml
# ✅ 修正後 — ステップは常に実行される
- name: Run large E2E tests
  if: always()
  run: npm run test:e2e:large
  env:
    RUN_VISUAL_TESTS: 1
    PLAYWRIGHT_BASE_URL: http://localhost:4173
```

「Upload Playwright report」ステップはすでに `if: always()` を持っていたため、失敗時もレポートはアップロードされていました。この修正により大規模 E2E ステップも同じ意図に沿うようになりました。

---

## 検証

すべての修正を適用した後：

```bash
cd frontend && npm run lint       # 0 エラー
cd backend  && npm run lint       # 0 エラー
cd frontend && npm run typecheck  # 0 エラー
cd backend  && npm run typecheck  # 0 エラー
```

---

## まとめ

| # | ルール | ファイル | 修正 |
|---|---|---|---|
| 1 | `import/order` | `UserProfilePage.tsx` | アルファベット順にインポートを並び替え（`jotai` を `react` の前に） |
| 2 | `no-misused-promises` | `UserProfilePage.tsx` | async ハンドラーを `.catch()` ラムダでラップ |
| 3 | `vitest/no-conditional-expect` | `user-profile-validation.small.test.ts` | `if` ガードを `assert()` + 無条件 `expect()` に置き換え |
| 4 | `stylistic/semi` | 両方のバックエンドテストファイル | ESLint 自動修正（`--fix`）でセミコロンを追加 |
| 5 | CI ステップの依存 | `ci-nightly.yml` | 大規模 E2E ステップに `if: always()` を追加 |

### 重要な教訓

- **`no-misused-promises`** は外観上のルールではなく安全性のルールです。`void` 型のイベント prop に `async` 関数を直接渡すと、未処理の rejection が隠れる可能性があります。常に同期ラムダでラップしてください。
- **`vitest/no-conditional-expect`** はサイレントなテストパスを防ぎます。テストで判別ユニオンメンバーにアクセスする前に型絞り込みガードとして Vitest の `assert()` を使用してください。
- **CI の `if: always()`** は独立したテストスイートが常に実行されることを保証し、前のステップが失敗した場合でもビルドの健全性の完全な画像を提供します。

**影響を受けるファイル:**

|ファイル |ルール違反 |
|---|---|
| `frontend/src/features/auth/pages/UserProfilePage.tsx` | `import/order`、`no-misused-promises` |
| `backend/src/infrastructure/user-profile-validation.small.test.ts` | `vitest/no-conditional-expect`、`stylistic/semi` |
| `backend/src/tests/user-profile.medium.test.ts` | `stylistic/semi` |
| `.github/workflows/ci-nightly.yml` |前のステップが失敗した場合にステップがスキップされる |

---

## 修正 1 — `import/order`: UserProfilePage.tsx の間違ったインポート順序

### エラー

```
import/order: 'jotai' import must come before 'react'
```

### 原因

プロジェクトの ESLint 構成 (`eslint-plugin-import` 経由) により、外部パッケージのインポートが強制されます。
アルファベット順にソートされています。元のコードでは、`jotai` の前に `react` がリストされています。

```ts
// ❌ Before
import { useState } from 'react'
import { useAtom } from 'jotai'
```

`j` は `r` の前にあるため、`jotai` が最初に現れる必要があります。

### Fix

2 つの行を入れ替えます。

```ts
// ✅ After
import { useAtom } from 'jotai'
import { useState } from 'react'
```

実行時の動作は変更されません。これは、インポートをスキャン可能に保つための純粋に表面的なルールであり、
コードベース全体で一貫性があります。

---

## 修正2 — `no-misused-promises`: 非同期関数が`onSubmit`に直接渡される

### エラー

```
no-misused-promises: Promise-returning function provided to attribute where a void return was expected
```

### 原因

`handleSubmit` は `async` 関数であり、`Promise` を返すことを意味します。 Reactの`onSubmit`
prop は、`void` を返すハンドラーを予期します。非同期関数を割り当てると、これに直接違反します。
契約：

```tsx
// ❌ Before — handleSubmit returns Promise<void>, not void
<form onSubmit={handleSubmit}>
```

`Promise` が予期せず拒否した場合、その拒否は React によって静かに飲み込まれます。
TypeScript/ESLint ルール `@typescript-eslint/no-misused-promises` は、表面化するように設計されています。
まさにこのクラスのバグです。

### Fix

非同期呼び出しを同期ラムダでラップし、`.catch()` をチェーンしてエラー処理を明示します。

```tsx
// ✅ After
<form onSubmit={(e) => { handleSubmit(e).catch(() => { /* handled inside */ }) }}>
```

コメント `/* handled inside */` は、エラー処理がすでに `handleSubmit` 内に存在していることを示します。
それ自体 (`setError` を呼び出す `try/catch` ブロックがあります)。 `.catch()` ラッパーは純粋にそこにあります
`void` の戻り要件を満たし、未処理の拒否警告が表示されないようにします。

---

## 修正 3 — `vitest/no-conditional-expect`: テストでの `expect()` の周りのガード

### エラー

```
vitest/no-conditional-expect: Expect must not be called in a conditional
```

### 原因

`user-profile-validation.small.test.ts` のいくつかのテストでは、アクセスする前に `if` ガードが使用されていました。
差別された組合員:

```ts
// ❌ Before — expect() is inside an if-block
const result = parseUpdateUserProfileInput(body)
if (result.success) {
  expect(result.email).toBe('valid@example.com')
}
```

ここでの問題は 2 つあります。

1. **ルール違反**: `vitest/no-conditional-expect` は内部での `expect()` 呼び出しを禁止します
条件が `false` の場合、アサーションは警告せずにスキップされるためです。
テストに失敗した。 `result.success` が `false` の場合でも、テストは合格します。
2. **TypeScript の絞り込みが失われます**: `if` ブロックの後、`result` は絞り込まれなくなりました。
success ブランチなので、`result.email` のようなプロパティは、その外ではタイプ セーフではありません。

このパターンはファイル全体で 6 回出現し、`email`、`currentPassword`、および `email` のケースをカバーしています。
`newPassword` フィールド アサーション。

### Fix

Vitest から `assert` をインポートし、`expect()` の前に無条件で呼び出します。

```ts
// ✅ After — assert() throws if false, so expect() is always reached
import { assert, describe, expect, it } from 'vitest'

const result = parseUpdateUserProfileInput(body)
assert(result.success)               // throws (fails the test) if false
expect(result.email).toBe('valid@example.com')  // now type-safe and always executed
```

`assert(result.success)` は型ガードとして機能します。これが合格すると、TypeScript は `result` が
後続のすべての行に成功分岐が表示されます。失敗した場合、テストは直ちに失敗し、クリアされます。
黙って渡すのではなく、メッセージを送ります。

`if (result.success) { expect(...) }` パターンの 6 つの出現すべてがこれに置き換えられました。
2行形式。

---

## 修正 4 — `stylistic/semi`: バックエンド テスト ファイルでセミコロンが欠落している

### エラー

```
stylistic/semi: Missing semicolon
```

### 原因

バックエンド ESLint 構成では、末尾のセミコロン (`stylistic/semi`) が強制されます。両方
`user-profile-validation.small.test.ts` および `user-profile.medium.test.ts` は、なしで書き込まれました。
ステートメントの末尾にセミコロンを使用します。これは、レッド キャンペーン中に使用されたスタイルと一致しています。
(テストの失敗) フェーズとグリーン (実装) フェーズがありましたが、強制されたルールからは逸脱していました。

### Fix

ESLint の自動修正プログラムを実行します。

```bash
npx eslint --fix backend/src/infrastructure/user-profile-validation.small.test.ts
npx eslint --fix backend/src/tests/user-profile.medium.test.ts
```

これにより、1 回のパスで両方のファイルのすべてのステートメントにセミコロンが追加されました。ロジックは変更されませんでした。

---

## 修正 5 — 夜間の CI: 以前の障害で大規模な E2E ステップがスキップされる

### 問題

`.github/workflows/ci-nightly.yml` では、「大規模な E2E テストを実行する」ステップに `if:` 条件がありませんでした。
GitHub Actions は、前のステップが失敗すると後続のステップをスキップします。そのため、「中程度の E2E テストを実行する」
ステップが失敗したため、大規模な E2E テストはまったく実行されず、潜在的な失敗がその中に隠されていました。
スイート。

```yaml
# ❌ Before — step is skipped if medium E2E step fails
- name: Run large E2E tests
  run: npm run test:e2e:large
```

### Fix

`if: always()` を追加して、前のステップの結果に関係なくステップを強制的に実行します。

```yaml
# ✅ After — step always runs
- name: Run large E2E tests
  if: always()
  run: npm run test:e2e:large
  env:
    RUN_VISUAL_TESTS: 1
    PLAYWRIGHT_BASE_URL: http://localhost:4173
```

「Playwright レポートのアップロード」ステップにはすでに `if: always()` が設定されているため、レポートは
失敗してもアップロードされます。この修正により、大規模な E2E ステップが同じ意図に沿ったものになります。

---

## 検証

すべての修正を適用した後:

```bash
cd frontend && npm run lint       # 0 errors
cd backend  && npm run lint       # 0 errors
cd frontend && npm run typecheck  # 0 errors
cd backend  && npm run typecheck  # 0 errors
```

---

## まとめ

| # |ルール |ファイル |修正 |
|---|---|---|---|
| 1 | `import/order` | `UserProfilePage.tsx` |インポートをアルファベット順に並べ替えました (`jotai` の前に `react`)。
| 2 | `no-misused-promises` | `UserProfilePage.tsx` | `.catch()` ラムダでラップされた非同期ハンドラー |
| 3 | `vitest/no-conditional-expect` | `user-profile-validation.small.test.ts` | `if` ガードを `assert()` + 無条件 `expect()` に置き換えました。
| 4 | `stylistic/semi` |両方のバックエンド テスト ファイル | ESLint 自動修正 (`--fix`) により欠落しているセミコロンが追加されました。
| 5 | CI ステップの依存関係 | `ci-nightly.yml` | `if: always()` を大規模な E2E ステップに追加しました。

### 重要なポイント

- **`no-misused-promises`** は安全ルールであり、表面的なルールではありません。 `async` 関数を直接渡す
`void` 型のイベント プロパティに追加すると、未処理の拒否が非表示になることがあります。常に同期でラップする
ラムダ。
- **`vitest/no-conditional-expect`** はサイレント テスト パスを防止します。 Vitest の `assert()` を
テストで区別された共用体メンバーにアクセスする前に型を絞り込むガード。
- **CI の `if: always()`** は、独立したテスト スイートが常に実行されることを保証し、
初期のステップが失敗した場合でも、ビルドの健全性の全体像を把握できます。
