# Storybookが成功しても本番ビルドが失敗した理由

## 結論

Storybookの例示コンポーネントをビルド・テストできても、通常アプリのエントリーポイントをビルドできるとは限りませんでした。今回のCIには `bun run build` を追加し、index.htmlからの本番用バンドルを検査するようにしました。

## 発生した問題

2026年10月22日のblog-wordcloudで確認しました。frontendはReact 19、Vite 8.3.4、Storybook 10.6.3です。

Storybookは3ファイル8テストが成功していました。一方、通常アプリのApp.tsxは、存在しないhero.png・react.svg・vite.svgをimportしていました。レビュー時のファイル出力を行わないViteビルドでは、3件のUNRESOLVED_IMPORTを確認しました。

```text
index.html → src/main.tsx → src/App.tsx → 存在しない画像
```

Storybookの設定はsrc以下のMDXとstoriesを入口にしており、このApp.tsxを読み込むstoryはありませんでした。

## 調査

GitHubのレビューでも、本番ビルドをCIへ追加する指摘がありました。型チェック・Lintと、Storybookの静的ビルド・ブラウザテストだけでは、production Viteの入口やCSS処理を実行したことにはならないためです。

型エラーが先に出る場合もあるので、失敗箇所を分けて確認しました。ユーザーの画像参照修正後は、残っていたHeaderの未使用React importでtscが停止しました。Headerの未使用importも除去し、本番ビルドを再確認しました。

## 解決方法

ユーザーが行った修正を確認し、欠落画像の参照と、それを使うスターター表示を削除しました。残る未使用importも除去したうえで、Frontend checksに次のstepを追加しました。

```yaml
- name: Build
  run: bun run build
```

このjobの既定のworking-directoryはapp/frontendです。そこで実行するbuild scriptは `tsc -b && vite build` なので、型チェックに加えて通常アプリのproduction bundleを作成します。

Storybookのbuild/tests jobも残しました。それぞれが読むコードの入口が違うためです。

## 動作確認

| 検証 | 結果 |
| --- | --- |
| frontendでbun run build | 成功、distへ本番成果物を生成 |
| frontendでbun run typecheck | 成功 |
| Storybookブラウザテスト | 3ファイル8テスト成功 |
| GitHub ActionsのBuild・Type check step | 成功 |
| GitHub ActionsのESLint step | 既存5エラーで失敗 |

対象の[Frontend CI実行](https://github.com/kishimin/blog-wordcloud/actions/runs/39776488863)では、Storybook build/testsも成功しました。CI全体が成功したとは記録していません。

## 再発防止と制約

本番ビルドの追加が今回実装した検査です。欠落画像の参照など、通常アプリ側のバンドル失敗をCIで検出できるようになりました。

これはアプリをブラウザで操作したE2Eテストではありません。実APIとの結合、VRT、既存Lint違反の解消は確認対象外です。

## 根拠

画像参照修正は997ed8b、CI追加は7fee2d2です。[CI追加のコミット](https://github.com/kishimin/blog-wordcloud/commit/7fee2d2d8806414428771668eb184f73a41d1379)と、App.tsx・package.json・Storybook設定、ローカル結果、上記CIを照合しました。
