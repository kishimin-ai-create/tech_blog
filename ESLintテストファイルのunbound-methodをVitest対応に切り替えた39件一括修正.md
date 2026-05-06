# ESLint テストファイルの `unbound-method` を Vitest 対応に切り替えた 39 件一括修正

## 対象読者

- TypeScript + Vitest + ESLint を使っているバックエンドエンジニア
- `@typescript-eslint/unbound-method` が Vitest テストファイルで大量に出て困っている人

---

## 問題の背景

バックエンドに新しいファイル群（mysql リポジトリ、マイグレーションランナーなど）を追加した後、`npm run lint` を実行すると 39 件のエラーが出た。エラーの内訳は次の4種類。

1. `@typescript-eslint/unbound-method` — テストファイルで `expect(obj.method).toHaveBeenCalled()` を呼んでいる箇所
2. `@typescript-eslint/no-unsafe-assignment` — テストファイルで型安全でない代入が検出された箇所
3. 関数宣言への不要な `async` キーワード（`await` を使っていない非同期関数）
4. 未使用 import

---

## 原因と修正

### 1. `unbound-method` — `@typescript-eslint` → `vitest` ルールに切り替え

`@typescript-eslint/unbound-method` はクラスメソッドをオブジェクトから切り離した形（unbound）で参照すると警告するルール。しかし Vitest の `expect(obj.method).toHaveBeenCalled()` パターンはまさにこの形になるため、テストファイルで大量に発火する。

このケースには `vitest/unbound-method` が正しい代替ルール。Vitest の mock パターンを理解した上で検査するため、偽陽性が出ない。

```typescript
// eslint.config.mts
{
  files: [
    "**/*.test.{ts,tsx}",
    "**/*.spec.{ts,tsx}",
    "**/__tests__/**/*.{ts,tsx}",
    "**/tests/**/**/*.{ts,tsx}",
  ],
  rules: {
    ...vitest.configs.recommended.rules,
    "@typescript-eslint/unbound-method": "off",   // ← 無効化
    "vitest/unbound-method": "error",             // ← Vitest 対応版を有効化
  },
}
```

設定のポイントは「テストファイル専用のオーバーライドブロックでのみ切り替える」こと。本番コードでは `@typescript-eslint/unbound-method` が有効なまま。

### 2. `no-unsafe-assignment` — テストファイルで無効化

テストファイルでは `vi.fn()` や `vi.mocked()` の戻り値など、型情報が弱い mock オブジェクトを扱うことが多い。`@typescript-eslint/no-unsafe-assignment` がこれらを全件エラーにするため、テストファイルのみ無効化した。

```typescript
{
  files: ["**/*.test.{ts,tsx}", /* ... */],
  rules: {
    "@typescript-eslint/no-unsafe-call": "off",
    "@typescript-eslint/no-unsafe-assignment": "off", // ← 追加
  },
}
```

本番コードへの影響はなく、テストの可読性を下げるルールのみをスコープを絞って切っている。

### 3. 不要な `async` キーワード除去

`await` を含まない関数に `async` が付いているケース。Vitest のテストヘルパー関数などで発生していた。

```typescript
// 修正前
async function setupFixtures() {
  return createApp({ name: 'test-app' });
}

// 修正後
function setupFixtures() {
  return createApp({ name: 'test-app' });
}
```

### 4. 未使用 import 削除

`unused-imports/no-unused-imports` ルールで検出された使われていない import を削除。リファクタリング中に残ったもの。

---

## 最終的な ESLint テストオーバーライド設定

```typescript
{
  files: [
    "**/*.test.{ts,tsx}",
    "**/*.spec.{ts,tsx}",
    "**/__tests__/**/*.{ts,tsx}",
    "**/tests/**/**/*.{ts,tsx}",
  ],
  rules: {
    ...vitest.configs.recommended.rules,
    "@typescript-eslint/no-unsafe-call": "off",
    "@typescript-eslint/no-unsafe-assignment": "off",
    "@typescript-eslint/unbound-method": "off",
    "vitest/unbound-method": "error",
    "vitest/max-nested-describe": ["error", { max: 3 }],
    "vitest/no-focused-tests": "error",
    "vitest/no-disabled-tests": "warn",
  },
  settings: {
    vitest: { typecheck: true },
  },
  languageOptions: { globals: { ...vitest.environments.env.globals } },
},
```

---

## まとめ

| エラー | 対処 |
|--------|------|
| `@typescript-eslint/unbound-method`（テストファイル） | `"off"` にして `vitest/unbound-method` に切り替え |
| `@typescript-eslint/no-unsafe-assignment`（テストファイル） | テストファイル限定で `"off"` |
| 不要な `async` キーワード | 該当関数から `async` を削除 |
| 未使用 import | 削除 |

`@typescript-eslint/unbound-method` と `vitest/unbound-method` の関係は Vitest 公式ドキュメントにも記載がある。Vitest プロジェクトでは ESLint 設定にテストファイル専用のオーバーライドブロックを設けて、この切り替えをデフォルトで入れておくと後から詰まらなくて済む。
