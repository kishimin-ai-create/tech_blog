# ESLint 品質改善：型エラー修正から eslint-disable 説明強制まで

## はじめに

`/// <reference types="..." />` はモジュール解決が整備されていなかった時代の回避策だ。`vitest/config` はそれ自体が `defineConfig` を export している ESM モジュールなので、普通に `import` すればトリプルスラッシュは不要になる。

## 対象読者

TypeScript + ESLint の flat config を使いはじめたエンジニア。特にモノレポで frontend（Next.js）と backend（Hono/Bun）を同時に整備している開発者。

---

## 背景

`diary` モノレポ（frontend: Next.js / backend: Hono + Bun）では ESLint flat config を使ってコード品質を管理している。プロジェクト初期セットアップ後に lint・型チェックを通してみると、いくつかのエラーと「とりあえず黙らせた」suppression コメントが残っていた。

今回は以下の 4 つの問題を解消し、最後に「説明なき eslint-disable 禁止」ルールを両 workspace に追加した。

1. `vitest.config.ts` のトリプルスラッシュ参照
2. 型定義を持たない ESLint プラグイン（TS7016 エラー）
3. axios のデフォルトインポートによる lint 警告
4. Next.js フォント変数の camelcase 警告

---

## 問題 1：vitest.config.ts のトリプルスラッシュ参照

### エラー

```
@typescript-eslint/triple-slash-reference: Do not use a triple slash reference for "vitest/config", use `import` style instead.
```

### 修正前

```ts
/// <reference types="vitest/config" />
import { defineConfig } from "vite";
```

### 修正後

```ts
import { defineConfig } from "vitest/config";
```

### なぜ問題だったか

`/// <reference types="..." />` はモジュール解決が整備されていなかった時代の回避策だ。`vitest/config` はそれ自体が `defineConfig` を export している ESM モジュールなので、普通に `import` すればトリプルスラッシュは不要になる。

`@typescript-eslint` はデフォルトで `triple-slash-reference` を警告・エラー扱いにしている。モダンな TypeScript + ESM 環境でトリプルスラッシュを使い続けると「なぜ通常 import で解決しないのか」という疑問が残り、可読性を損なう。

### おまけ：`passWithNoTests: true` の追加

同じファイルにもう 1 つ変更を入れた。

```ts
test: {
  passWithNoTests: true,   // 追加
  // ...
}
```

テストファイルがまだ存在しない状態で `bun run test` を走らせると Vitest はデフォルトで exit code 1 を返す。CI の初期フェーズでは「テストが 0 件 = 正常」という状態が続くため、このオプションで CI 失敗を防いでいる。

---

## 問題 2：型定義のない ESLint プラグインによる TS7016

### エラー

```
TS7016: Could not find a declaration file for module 'eslint-plugin-drizzle'.
TS7016: Could not find a declaration file for module 'eslint-plugin-security'.
```

`backend/eslint.config.mts` は `.mts` 拡張子の TypeScript ファイルなので、型チェックの対象になる。`eslint-plugin-drizzle` と `eslint-plugin-security` は TypeScript の型定義（`@types/xxx` または同梱の `.d.ts`）を持っていないため、import 時にエラーになる。

### 対処：アンビエントモジュール宣言

```ts
// backend/src/types/eslint-plugins.d.ts
declare module "eslint-plugin-drizzle";
declare module "eslint-plugin-security";
```

`declare module "xxx"` は「このモジュールは存在するが型情報はない。`any` として扱え」という意味のアンビエント宣言だ。`@types/xxx` を別途メンテするより軽量で、ESLint プラグインのように「型が不要な設定専用モジュール」には十分な対処法といえる。

> **注意点**：アンビエント宣言は型安全性を捨てる選択でもある。利用箇所が設定ファイルのみで、プロダクションコードで直接触らない場合に限り使うのが望ましい。`eslint.config.mts` 自体も `@typescript-eslint/no-unsafe-assignment` などを `off` にして型チェックを緩和している。

---

## 問題 3：axios のデフォルトインポートで発生する lint 警告

### 経緯

最初のコミット（`fix: resolve lint errors`）では、次のように一時的に suppression コメントで黙らせた。

```ts
// eslint-disable-next-line import/no-named-as-default-member
const axiosInstance = axios.create({ baseURL: BACKEND_URL });
```

`import/no-named-as-default-member` はデフォルトインポートのメンバーアクセス（`axios.create`）を警告するルールだ。axios の型定義では `create` は名前付きエクスポートとして存在するので、`axios.create()` という呼び出し方は「デフォルトオブジェクトのプロパティ」とみなされ警告になる。

### 本来の修正：名前付きインポートに変更

```ts
// Before
import axios from "axios";
const axiosInstance = axios.create({ baseURL: BACKEND_URL });

// After
import { create } from "axios";
const axiosInstance = create({ baseURL: BACKEND_URL });
```

次のコミット（`chore: remove unnecessary files and improve axios import`）で suppression を取り除き、名前付きインポートへ切り替えた。

**suppression でごまかす vs 正しく直す**の違いが明確に現れた例だ。lint ルールが警告を出しているとき、それはコードの書き方を変えることで解消できる場合が多い。まず「なぜ警告が出ているか」を理解してから suppression を使うかどうか判断すべきだ。

---

## 問題 4：Next.js フォント変数の camelcase 警告

### 状況

```ts
import { Geist, Geist_Mono } from "next/font/google";
```

`Geist_Mono` は Next.js が snake_case で export している変数名であり、開発者側では変更できない。`camelcase` ルールが警告を出すが、これは**仕様上の例外**なので suppression で対応するのが正しい。

### eslint-disable-next-line に説明を付けた最終形

```ts
// eslint-disable-next-line camelcase -- Geist_Mono is a Next.js font variable exported in snake_case by design
import { Geist, Geist_Mono } from "next/font/google";
```

この `-- Geist_Mono is ...` の部分こそが、次のセクションで導入したルールによって**必須**になった説明だ。

---

## 新ルール：eslint-disable に説明を必須化

### 課題

eslint-disable コメントは便利だが、説明なしだと「なぜここで無効化したのか」が後から分からない。レビューで見逃されたまま蓄積されると、何年後かに誰も理由を知らない suppression だらけのコードになる。

### 導入したプラグインとルール

```
@eslint-community/eslint-plugin-eslint-comments@4.7.2
```

`require-description` ルールを `"error"` に設定した。

```ts
// 違反例（CI で落ちる）
// eslint-disable-next-line camelcase

// 適合例（CI を通る）
// eslint-disable-next-line camelcase -- Geist_Mono は Next.js が snake_case で export している
```

### frontend への追加（`eslint.config.mjs`）

```ts
import eslintComments from "@eslint-community/eslint-plugin-eslint-comments";

export default defineConfig([
  {
    plugins: {
      "unused-imports": unusedImports,
      "@eslint-community/eslint-comments": eslintComments,
    },
    rules: {
      "@eslint-community/eslint-comments/require-description": "error",
      // ...
    },
  },
  // ...
]);
```

### backend への追加（`eslint.config.mts`）

```ts
import eslintComments from "@eslint-community/eslint-plugin-eslint-comments";

export default defineConfig([
  {
    plugins: {
      // ...
      "@eslint-community/eslint-comments": eslintComments,
    },
    rules: {
      "@eslint-community/eslint-comments/require-description": "error",
    },
  },
  // ...
]);
```

どちらも **グローバル設定ブロック**に追加したため、ファイルの種類を問わずすべてのコードに適用される。

### なぜ `@eslint-community/` なのか

もともと `eslint-plugin-eslint-comments` という名前で公開されていたプラグインが、ESLint Community org に移管されてスコープ付きパッケージになったものだ。旧パッケージは ESLint v9 flat config に未対応のため、`@eslint-community/` 版を使う必要がある。

---

## 変更の全体像

| ファイル | 変更内容 |
|---|---|
| `frontend/vitest.config.ts` | トリプルスラッシュ削除・`import from "vitest/config"`・`passWithNoTests: true` 追加 |
| `frontend/app/layout.tsx` | `// eslint-disable-next-line camelcase` に `-- 理由` を追記 |
| `frontend/app/api/mutator/custom-instance.ts` | `import axios` → `import { create }` に変更し suppression を削除 |
| `backend/src/types/eslint-plugins.d.ts` | 型なし ESLint プラグインへのアンビエント宣言を新規作成 |
| `frontend/eslint.config.mjs` | `eslint-comments` プラグイン + `require-description: "error"` を追加 |
| `backend/eslint.config.mts` | 同上 |

---

## まとめ

今回の改善を通じて、次の 3 つの考え方が整理できた。

1. **suppression より修正を優先する**：`import/no-named-as-default-member` のように、書き方を変えれば suppression 不要になるケースは多い。警告の意味を理解してから判断する。

2. **仕方ない例外には必ず理由を書く**：Next.js の `Geist_Mono` のように外部ライブラリの命名規則に由来する suppression は正当だが、`-- 理由` がないと後から判断できない。`require-description` はこれをコードベース全体で機械的に強制する。

3. **型定義のないプラグインにはアンビエント宣言**：ESLint 設定ファイルを `.mts` で書く場合、`@types/xxx` が存在しないパッケージは `declare module "xxx"` で明示的に型なしとして宣言し、TS7016 エラーを解消する。
