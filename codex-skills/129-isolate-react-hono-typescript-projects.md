# ReactとHonoの型チェックをパッケージごとに分離する

## はじめに

同じBun workspaceでも、React frontendとHono backendには異なるJSX設定が必要でした。今回の変更では、共通設定を小さく保ち、各パッケージが自身の実行環境を宣言する形へ分けました。

## 前提・環境

2026年10月21日に確認したblog-wordcloudの `ef04f2d` 以降の構成です。実行環境はWindows、Bun 1.3.13です。この記事はパッケージの検査範囲を扱い、TypeScript APIの版の問題は[別記事](./130-typescript-compiler-and-eslint-api-versions.md)で扱います。

## やってみた結果

`app` にbackend・frontend・schemasのworkspaceを定義し、それぞれへpackage.jsonと型検査設定を配置しました。共通のtsconfig.base.jsonにはstrict、skipLibCheck、noEmitだけを置きました。

| 対象 | 設定の責任 | 検査コマンド |
| --- | --- | --- |
| backend | Bunの型、Hono JSX、srcと共有env型 | `tsc --noEmit -p tsconfig.json` |
| frontend | React JSX、DOM、Viteの型、設定ファイル | `tsc -b` |
| schemas | スキーマ実装と依存パッケージ | `tsc --noEmit -p tsconfig.json` |

## 実装・検証

### コンパイラーの引数をプロジェクト構造に合わせる

backendの設定は `src/**/*.ts`、`src/**/*.tsx` と `../types/**/*.ts` をincludeします。`-p` の検査対象は、そのtsconfigが定義するプロジェクトです。

frontendのルートtsconfig.jsonはfilesが空で、app用とNode設定用の2つの設定をreferencesに持ちます。ここでは `tsc -b` を使います。backend用のコマンドをfrontendのsolution設定へそのままコピーして、同じ範囲が検査されるとは考えませんでした。

### 個別に結果を確認する

以下は `app` で実行しました。

```powershell
bun run --cwd backend typecheck
bun run --cwd schemas typecheck
bun run --cwd frontend typecheck
```

backendとschemasは成功しました。frontendはButton.tsxとHeader.tsxの未使用React importで失敗しました。設定を分けたことと、すべてのソースが検査に合格することは別の結果として記録しています。

## つまずいたところ

ルートの `bun run typecheck` はbackendとschemasだけを実行します。frontendを含めるコマンドは `bun run typecheck:all` です。コマンド名だけから「全パッケージ成功」と読み替えると検査範囲を誤ります。

## 学んだこと

`tsc --noEmit -p tsconfig.json` が妥当かどうかは、そのファイルのinclude・referencesと期待する対象から判断します。共通設定を共有しても、各パッケージの実行環境と検査対象は明示する必要がありました。

## 根拠と制約

根拠は `ef04f2d` の差分、各package.json、tsconfigと個別実行結果です。[対象コミット](https://github.com/kishimin/blog-wordcloud/commit/ef04f2d8c7076389c2368f3ecec1a5afbbfe6184)で設定を確認できます。アプリケーション画面のビルド成功を示す記事ではありません。
