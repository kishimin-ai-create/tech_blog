# Storybookのビルド成功後に、未配置のMSW Workerで表示が止まった

## 結論

profile-hubのStorybookはビルドに成功したが、実際のiframeではMSWのWorker登録に失敗してstoryを表示できなかった。共通previewがWorkerを初期化する一方、配信先に`mockServiceWorker.js`がなかった。今回はすべて静的なstoryだったため、不要な共通MSW loaderを外して表示を確認した。

2026年10月27日、Storybook 10.6.0での作業記録である。対象PR群は未マージ。ビルドと実行の確認範囲一般については[既存の記事156](156-storybook-success-and-production-build.md)を参照し、ここではWorker登録の失敗に絞る。

## 症状と調査

`apps/public-web`で`bun run build-storybook`が成功した後、生成物を配信してブラウザで開いた。iframeでは`mockServiceWorker.js`の登録に失敗し、JavaScriptとして期待する応答が`text/html`になっていた。

previewには`initialize`と`mswLoader`が設定されていた。しかし、期待するWorkerファイルは配置されていなかった。ブラウザが要求するファイルと、実際の配信物を突き合わせることで、アプリのコンポーネントを表示する前に失敗していることが分かった。

## 静的storyに合わせて初期化を外した

今回のstoryはEngineering・HobbyのHeroと、QueryStateの状態別表示だった。HTTP通信を必要とせず、表示用の値を直接渡している。このため、previewから共通のMSW初期化とloaderを外した。

MSW全体が不要という判断ではない。HTTPを利用するstoryがある場合は、必要なWorkerを正しく配信し、handlerとloaderを構成する方針を採る必要がある。今回は不要な実行時依存が全storyを止めていたことに対する修正だった。

## 動作確認

修正後にStorybookを再ビルドし、ビルド済みの8 storyをブラウザで開いた。Hero 2件とQuery状態6件が表示できることを確認した。ビルド成功に加えて、配信されたiframeの実行まで確認した点が重要である。

## 再発を防ぐ確認

共通previewに処理を加えると、個別のstoryが使わない処理でもすべてに影響する。新しく初期化を追加するときは、必要な配信ファイルと、通信を使わないstoryへの影響を一緒に確認したい。

今回、MSWを使うstoryの自動検証を追加したわけではない。将来追加する場合はWorkerの配信と実際の応答まで確認する作業が残る。

## 根拠

- [対象コミットのpreview](https://github.com/kishimin-ai-create/profile-hub/blob/4457f6f8624782d80bb8196a4c00c5319cf24325/apps/public-web/.storybook/preview.ts)
- [PR #44：Hero](https://github.com/kishimin-ai-create/profile-hub/pull/44)、[PR #49：Query状態story](https://github.com/kishimin-ai-create/profile-hub/pull/49)
