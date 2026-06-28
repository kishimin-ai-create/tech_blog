# ESLint の `no-undef` が `process.env` で誤検知する原因と修正方法 — Flat Config の globals 設定漏れ

## エラー概要

`frontend/` で `bun run lint` を実行すると、`playwright.config.ts` や `vitest.config.ts` といったツール設定ファイルの `process.env` 参照に対して ESLint の `no-undef` エラーが発生していた。

```
error  'process' is not defined  no-undef
```

アプリケーションコード（`*.tsx` / `*.ts`）では同じ `process.env` を使っても問題ないのに、設定ファイルだけエラーになる——という一見不思議な挙動だった。

---

## 原因

### ESLint Flat Config における `globals` の継承と上書き

ESLint の新しい Flat Config（`eslint.config.mjs`）では、各ブロックが **独立したオーバーライドとして積み重なる** 仕組みになっている。`languageOptions.globals` はブロックごとに完全に上書きされるのではなく、後から合成されるが、**同じキーが存在する場合は後のブロックが勝つ**。

問題が起きていた設定を整理すると、次のような構成になっていた。

```mjs
// ブロック①: TypeScript + React ファイル全般
{
  files: ["**/*.{ts,tsx}"],
  languageOptions: {
    globals: globals.browser,   // ← ブラウザグローバルを宣言
    // ...
  },
},

// ブロック②: 設定・ツールファイル用オーバーライド（修正前）
{
  files: ["*.config.{js,mjs,ts,mts}", ".storybook/**", "e2e/**", "vitest.setup.ts"],
  extends: [tseslint.configs.disableTypeChecked],
  // languageOptions が未指定
},
```

`*.config.ts` ファイルはブロック①の `files` パターン（`**/*.{ts,tsx}`）にも合致するため、ブラウザグローバルは一度宣言される。しかしブロック②は `languageOptions` を持たないため、Node.js グローバル（`process`、`__dirname` など）は **どのブロックでも宣言されていない** 状態になっていた。

つまり `process` という識別子の出どころが ESLint の解析スコープに存在せず、`no-undef` が発火していた。

### `tseslint.configs.disableTypeChecked` は globals を変更しない

型チェックルールを無効化する `disableTypeChecked` は、`@typescript-eslint/` 系のルールを `off` にするだけであり、`languageOptions.globals` には一切触れない。そのため、ブロック②を加えても Node.js グローバルが宣言されることはなかった。

---

## 修正

`languageOptions.globals` に `globals.node` を追加するだけで解決する。

```diff
 {
   files: ["*.config.{js,mjs,ts,mts}", ".storybook/**", "e2e/**", "vitest.setup.ts"],
   extends: [tseslint.configs.disableTypeChecked],
+  languageOptions: {
+    globals: globals.node,
+  },
 },
```

`globals.node` は `process`、`__dirname`、`__filename`、`Buffer`、`require` などの Node.js 組み込みグローバルを一括宣言する。設定ファイルや E2E テスト（Playwright）はブラウザ上ではなく Node.js 環境で実行されるため、`globals.node` を指定するのが適切な選択だ。

修正後のブロック全体は以下のとおり。

```mjs
// Disable type-checked rules for config/tooling files
{
  files: [
    "*.config.{js,mjs,ts,mts}",
    ".storybook/**",
    "e2e/**",
    "vitest.setup.ts",
  ],
  extends: [tseslint.configs.disableTypeChecked],
  languageOptions: {
    globals: globals.node,
  },
},
```

`bun run lint` はエラー 0 件で終了するようになった。

---

## 注意点

### globals はブロック単位で意識する

Flat Config では `languageOptions.globals` を省略すると「グローバルなし」ではなく「宣言しない（前のブロックからの合成を受けるだけ）」という状態になる。  
`files` パターンが複数のブロックにマッチするファイルがある場合、**そのファイルが実際に動く実行環境**（ブラウザ／Node.js／両方）に合わせて globals を明示することが重要だ。

| ファイル種別 | 適切な globals |
|---|---|
| ブラウザで動くアプリコード | `globals.browser` |
| Node.js で動く設定・ツールファイル | `globals.node` |
| Vitest テスト（jsdom 環境） | `vitest.environments.env.globals` |
| Service Worker / Edge Runtime | `globals.serviceworker` など |

### `no-undef` は TypeScript プロジェクトでも有効

TypeScript の型チェックが有効な場合、`process` は `@types/node` によって型定義されるため、`tsc` ではエラーにならない。しかし ESLint の `no-undef` ルールはあくまで **ESLint の globals 宣言** を参照するため、型定義とは独立して機能する。両者が見ている情報源が違う点を意識しておくと、今回のような誤検知の原因を素早く特定できる。

---

## まとめ

| 項目 | 内容 |
|---|---|
| **エラー** | `no-undef: 'process' is not defined` |
| **発生ファイル** | `*.config.ts` など設定・ツールファイル |
| **根本原因** | Flat Config の設定ファイル用オーバーライドブロックに `languageOptions.globals` が未指定で、Node.js グローバルが宣言されていなかった |
| **修正内容** | 対象ブロックに `languageOptions: { globals: globals.node }` を追加 |
| **教訓** | Flat Config では実行環境に合わせた `globals` を各ブロックで明示する |
