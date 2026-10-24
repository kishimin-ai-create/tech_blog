# 型情報を使うESLintルールとテストルールの対象範囲を揃える

## はじめに

frontendとbackendでESLintを共有した際、共通化するルールと、TypeScriptプロジェクトに属するファイルの範囲を分ける必要がありました。特にsrc外のテストでは、テスト用プラグインの型情報設定も確認しました。

## 前提・環境

2026年10月21日、Bun 1.3.13、ESLint 9、typescript-eslint 8系、Vitest ESLint pluginを使ったblog-wordcloudの設定です。

## やってみた結果

共通設定の生成関数へ各パッケージの `tsconfigRootDir` と実行環境のglobalsを渡します。型情報を使うrecommendedTypeCheckedは `src/**/*.{ts,tsx}` へ限定しました。frontendへReact・Hooks・アクセシビリティを追加し、backendにはBunの実行環境を設定しました。

## 実装・検証

### 型情報の利用範囲を宣言する

共通設定の要点は次です。実際の設定から対象範囲に関する部分だけを示します。

```javascript
{
  files: ["src/**/*.{ts,tsx}"],
  extends: [tseslint.configs.recommendedTypeChecked],
  languageOptions: {
    parserOptions: { projectService: true, tsconfigRootDir },
  },
}
```

型情報を要しないTypeScript設定側にもtsconfigRootDirを指定しました。複数のプロジェクトルートがある環境で、設定位置の推定へ依存しないためです。

### src外のテストも構文ルールで検査する

テストファイルには集中実行やdescribeの深さなどのルールを適用します。ただし、src外のテストでは `settings.vitest.typecheck` をfalseにしました。TypeScriptプロジェクトへ含まれないファイルで、プラグインだけが型情報を要求しないようにするためです。

backendでは `vitestImports: ["bun:test"]` を指定し、Bunのtest importを認識させました。

### 対象範囲の違いを確認する

一時的な違反例による15件の設定検証を行いました。うち4件はsrc外のテストです。集中実行指定などが検出され、型情報を取得できないために解析が止まる状態を解消したことを確認しました。一時ファイルは恒久的な回帰テストとして追加していません。

## つまずいたところ

型付きルールをsrcへ限定するだけでは、テストプラグイン側の型情報要求まで自動的に揃うわけではありませんでした。parser設定とplugin設定の両方に適用範囲がありました。

## 学んだこと

共通化の単位は「同じファイルへ適用できるルール」です。プロジェクト外のファイルを無条件に検査対象から外す前に、型情報なしで維持できるルールを分けると、構文検査を残せました。

## 根拠と制約

[共有ESLint設定](https://github.com/kishimin/blog-wordcloud/blob/f5c4845/app/eslint.config.base.mjs)とfrontend/backendの追加設定、15件のローカル検証が根拠です。通常のLintはfrontendで15エラー、backendで4エラーが残り、全体としては成功していません。
