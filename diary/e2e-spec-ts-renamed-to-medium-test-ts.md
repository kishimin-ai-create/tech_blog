# E2Eテストファイルを `example.spec.ts` から `example.medium.test.ts` にリネームした

## はじめに

このリポジトリには `.github/instructions/test.instructions.md` というテスト規約ファイルがあり、**すべてのテストファイル名にサイズプレフィックスを含めること**が義務付けられています。

## 対象読者

- このリポジトリのテストファイル命名規則を把握したいエンジニア
- `*.spec.ts` 形式のテストファイルを追加しようとしている人

---

## 背景：リポジトリには「サイズプレフィックス必須」のルールがある

このリポジトリには `.github/instructions/test.instructions.md` というテスト規約ファイルがあり、**すべてのテストファイル名にサイズプレフィックスを含めること**が義務付けられています。

| サイズ | 命名パターン | 具体例 |
|--------|-------------|--------|
| Small  | `*.small.test.ts(x)` | `createApp.small.test.ts` |
| Medium | `*.medium.test.ts(x)` | `home.medium.test.ts` |
| Large  | `*.large.test.ts(x)` | `checkout.large.test.ts` |

規約には次のように明記されています。

> **Never** create `*.test.ts`, `*.test.tsx`, `*.spec.ts`, or `*.spec.tsx` without the size prefix.

サイズは「ネットワークアクセスの有無」「DB使用の有無」「外部システム依存の有無」などで分類されます。E2E テストはローカルホストへのアクセスや複数スレッドを使うため、`medium` に分類されます。

---

## 問題：`example.spec.ts` はルール違反だった

Playwright のデフォルト生成ファイルとして置かれていた `frontend/e2e/example.spec.ts` は、サイズプレフィックスを持たない `*.spec.ts` 形式でした。これは上記ルールに違反しています。

ファイルの中身は以下のシンプルな E2E テストで、内容自体に問題はありませんでした。

```ts
// frontend/e2e/example.spec.ts（変更前）
import { expect, test } from "@playwright/test";

test("home page / on load / page title includes 'diary'", async ({ page }) => {
  await page.goto("/");
  await expect(page).toHaveTitle(/diary/i);
});
```

---

## 対応：`git mv` によるリネーム

ファイルの内容は変更せず、ファイル名だけを修正しました。

```bash
git mv frontend/e2e/example.spec.ts frontend/e2e/example.medium.test.ts
```

`git mv` を使ったのは **git の変更履歴を保持するため**です。単純にファイルを削除・作成すると履歴が切れてしまいますが、`git mv` であれば `git log --follow` で過去のコミットをたどれます。

---

## ESLint・型チェックへの影響を確認

リネーム後、ESLint 設定と型チェック設定への影響を確認しました。

### ESLint（`frontend/eslint.config.mjs`）

テストファイル向けのオーバーライドブロックは次のパターンを対象にしています。

```js
files: [
  "**/*.{small,medium,large}.test.{ts,tsx}",  // ← リネーム後のファイルがここにマッチ
  "**/*.test.{ts,tsx}",
  "**/*.spec.{ts,tsx}",
  ...
],
```

`example.medium.test.ts` は `**/*.{small,medium,large}.test.{ts,tsx}` にマッチするため、Testing Library プラグインと Vitest プラグインの両方が正しく適用されます。**設定変更は不要でした。**

### 型チェック除外設定（`disableTypeChecked` オーバーライド）

```js
files: [
  "*.config.{js,mjs,ts,mts}",
  ".storybook/**",
  "e2e/**",        // ← e2e/ 以下を一括で除外
  "vitest.setup.ts",
],
```

`e2e/**` というパターンがすでに存在しており、リネーム後も変わらずカバーされます。こちらも**設定変更は不要でした。**

### 検証結果

```
bun run lint       → エラー 0 件
bun run typecheck  → e2e とは無関係な .storybook/main.ts の既存エラーのみ
```

---

## まとめ

| 項目 | 変更前 | 変更後 |
|------|--------|--------|
| ファイル名 | `example.spec.ts` | `example.medium.test.ts` |
| ファイル内容 | 変更なし | 変更なし |
| ルール準拠 | ❌ サイズプレフィックスなし | ✅ `medium` プレフィックスあり |
| ESLint 設定 | 変更なし | 変更なし |
| 型チェック設定 | 変更なし | 変更なし |

今回の修正は一行のコードも書かずに済みましたが、**リポジトリのルールに沿ったファイル命名は、テストランナーや ESLint のパターンマッチングが正しく機能する前提**です。新しいテストファイルを追加するときは、必ずサイズプレフィックスを付けてください。

---

## 関連規約

- `.github/instructions/test.instructions.md` — テストサイズの分類基準と命名ルール
