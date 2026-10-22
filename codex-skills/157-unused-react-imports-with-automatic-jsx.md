# Storybookは動くのに未使用React importで本番ビルドが止まった

## 結論

JSXを使っているだけのButton.tsxとHeader.tsxでは、先頭のReact importを削除すると型チェックが通りました。このプロジェクトは `jsx: "react-jsx"` と `noUnusedLocals: true` を指定しており、未使用のReact識別子がTS6133の対象になっていました。

## 発生した問題

2026年10月22日、blog-wordcloudのfrontendで確認しました。TypeScriptは6.0.3、build scriptは `tsc -b && vite build` です。

```text
src/stories/Header.tsx(1,1): error TS6133:
'React' is declared but its value is never read.
```

最初のレビューではButtonとHeaderの2件を確認しました。その後、ユーザーがButtonを修正した状態でもHeaderが残り、本番buildの最初のtscで停止しました。

## 調査

tsconfig.app.jsonはsrcを検査対象にしているため、storiesのサンプルコンポーネントも型検査へ含まれます。

ButtonとHeaderはReactという識別子を参照していませんでした。対してPage.tsxはReact.FCとReact.useStateを使用しており、同じimportを使っています。一律にすべてのReact importを消す変更にはしませんでした。

Storybookのブラウザテストが成功していたことと、上記tscによる検査を通過することは別の観測結果です。

## 解決方法

未使用のButtonとHeaderだけから、次のimportを削除しました。

```diff
-import React from 'react';
```

noUnusedLocalsは無効にせず、Pageの使用中のimportも残しています。ユーザーが行ったButtonの修正に、Headerの残りを補完しました。

## 動作確認

app/frontendから実行しました。

```powershell
bun run build
bun run typecheck
bun run test:storybook:coverage --reporter=default --reporter=json --outputFile=coverage/test-result.json
```

本番ビルドと型チェックが成功し、Storybookの8テストも成功しました。GitHub ActionsでもBuild・Type checkの成功を確認しました。

## 再発防止と制約

今回のCIには本番buildを加えたため、今後のコンパイル失敗もそのstepで検出します。未使用importの文字列だけをassertするテストは追加していません。既存のコンパイラーによる検査を使いました。

関数カバレッジは66.66%で、80%には達していません。またPageのJSX表記とVite設定のimport順に関する既存Lint違反は残しています。

## 根拠

修正は90bfa0aです。Button/Header/Pageの実装、tsconfig.app.json、修正前後のbuild出力、[対象CI](https://github.com/kishimin/blog-wordcloud/actions/runs/39776488863)が根拠です。
