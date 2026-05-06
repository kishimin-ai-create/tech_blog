# `ecosystem.config.cjs` を TypeScript 型情報 ESLint ルールの対象外にしてCIクラッシュを修正した

## エラー概要

バックエンドの CI で `npm run lint` を実行すると、`ecosystem.config.cjs` の処理時に次のようなエラーが発生してクラッシュしていました。

```
error  Parsing error: "parserOptions.project" has been set for @typescript-eslint/parser.
The file does not match your project config:
  This rule requires type information.
```

ローカル（Windows）では再現せず、CI（ubuntu-latest）だけで確認される状況でした。

## 原因

`backend/eslint.config.mts` の設定構造に起因します。

```ts
// ① 型情報ありルールをグローバル（全ファイル対象）に適用
tseslint.configs.recommendedTypeChecked,

// ② parserOptions.projectService は TS ファイルだけに制限
{
  files: ["**/*.{ts,mts,cts}"],
  languageOptions: {
    parserOptions: {
      projectService: true,
    },
  },
},
```

`tseslint.configs.recommendedTypeChecked` には `@typescript-eslint/await-thenable` や `@typescript-eslint/no-floating-promises` といった **型情報を必要とするルール** が含まれています。これらは `parserOptions.projectService`（または `parserOptions.project`）が設定されていないファイルでは動作できません。

`parserOptions.projectService: true` は `**/*.{ts,mts,cts}` にしか適用していないため、`.cjs` ファイルである `ecosystem.config.cjs` は型情報なしで処理されます。その状態で型情報必須ルールが発火するため、ESLint がクラッシュします。

### なぜローカルでは出なかったか

ESLint はデフォルトでドット始まりではない `.cjs` ファイルを対象にします。ローカル環境では `eslint.config.mts` の `ignores` に `ecosystem.config.cjs` が含まれていなかったものの、何らかの理由でそのファイルが ESLint の処理対象に入らないケースがあったと考えられます（実行ディレクトリや glob の解決差異など）。CI 上の Linux 環境では確実にスキャン対象になり、クラッシュが顕在化しました。

## 修正

`eslint.config.mts` のグローバル `ignores` 配列に `ecosystem.config.cjs` を追加しました。

```diff
 {
-  ignores: ["coverage/**", "dist/**"],
+  ignores: ["coverage/**", "dist/**", "ecosystem.config.cjs"],
 },
```

**変更ファイル**: `backend/eslint.config.mts`（1行変更）

`ecosystem.config.cjs` は PM2 の起動設定ファイルであり、TypeScript ではなく CommonJS 形式で記述されています。型情報を必要とする ESLint ルールの対象にする意味はないため、ignores に加えるのが適切な対応です。

### TDD サイクルとの対応

```
RED   → npm run lint を実行 → CI と同じクラッシュを確認
GREEN → ignores に追加 → lint 通過
```

## まとめ

| 観点 | 内容 |
|------|------|
| 根本原因 | `recommendedTypeChecked` がグローバル適用されているが、`.cjs` ファイルには `parserOptions.projectService` が設定されていない |
| 修正方法 | `ecosystem.config.cjs` をグローバル `ignores` に追加 |
| 変更規模 | 1行 |
| 再発防止 | TypeScript 以外のファイル（`.cjs`、`.mjs` など）をプロジェクトに追加する際は、typed lint の対象外に明示することを確認する |

`tseslint.configs.recommendedTypeChecked` を使う場合、**型情報なしでは動けないルール群**がグローバルに展開されるため、TypeScript 管理外のファイルは必ず `ignores` で除外するか、専用の `files` スコープで typed lint を適用する範囲を絞ることが重要です。
