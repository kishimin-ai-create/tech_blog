# pi-lab 技術記事

React、Vite、Vitest、Storybook、Playwrightで構成された「割り切れない研究所」の開発から得た知見をまとめる。各記事はPlaywright 1.61.1での検証結果に基づき、確認日は各記事末尾に記載する。

## Visual Regression Testing

- [Playwrightで主要導線の5状態を画像比較する](./01-playwright-vrt-five-states.md)
- [Playwright Clockで時間依存VRTを決定的にする](./02-playwright-clock-deterministic-vrt.md)
- [PlaywrightでLinux用の基準画像がないエラーを解決する](./03-playwright-linux-snapshot-baselines.md)
- [Playwright Artifactから画像差分の原因を調べる](./04-playwright-artifact-image-diff-analysis.md)
- [PlaywrightのVRT環境をDockerで固定する](./05-pin-playwright-docker-in-ci.md)
- [VRTを全Playwrightプロジェクトへ拡張する](./06-vrt-across-all-playwright-projects.md)

## テスト設計とCI

- [Playwright Fixtureの上書きでPOMを配布する](./07-playwright-fixture-pom.md)
- [VitestからPlaywright E2Eを確実に除外する](./08-vitest-exclude-playwright-e2e.md)
- [Vitest Browser ModeのCIにChromiumを導入する](./09-vitest-browser-ci-chromium.md)
- [Playwrightの動画撮影設定を一時利用に限定する](./10-temporary-playwright-video-recording.md)
- [StorybookのInteraction Testが古いアセットパスを検証し続けていた話](./11-storybook-stale-asset-path-assertion.md)
- [VRT基準画像の更新で、Dockerのbind mountがWindowsのnode_modulesを壊した話](./12-docker-bind-mount-node-modules-corruption.md)

## 設計判断とレビューの学び

- [Page ObjectのLocatorは、増える前提なら引数付きメソッドにする](./13-locator-method-over-hardcoded-property.md)
- [壁時計に依存しない画面なら、E2EでPage Clockを固定しない](./14-stop-mocking-clock-without-wall-clock-rendering.md)
- [テスト設定を触る前に、公式ドキュメントで既定値と用法を確認する](./15-verify-defaults-against-official-docs.md)
- [実装がADRと矛盾したら、無視も強行もせず新しいADRで折り合いをつける](./16-reconcile-adr-conflict-with-new-adr.md)

## デプロイ

- [Cloudflareの「Pages」のつもりで用意した`_redirects`が、実際にはWorkers Buildsでデプロイ失敗の原因になった話](./17-cloudflare-pages-to-workers-redirects-failure.md)
- [Workerスクリプトを書かずに、`wrangler.jsonc`だけで静的SPAをCloudflare Workersへ配信する](./18-wrangler-assets-only-worker-config.md)
- [`wrangler deploy`と`wrangler versions upload`は別物——Workers Buildsのブランチ別デプロイコマンド](./19-wrangler-deploy-vs-versions-upload.md)
- [`wrangler deploy --dry-run`はログイン不要で試せるが、サーバー側の検証は再現しない](./20-validate-wrangler-config-with-dry-run.md)
- [新規プロジェクトの設定画面に、別プロジェクト名のビルドトークンが表示された](./21-shared-build-token-across-projects.md)

## 共通の前提

- Node.js 22または24
- Playwright 1.61.1
- Vitest 4.1.10
- Windows 11でのローカル実行と、Ubuntu NobleコンテナでのCI相当検証

バージョン更新後は、各記事のコマンドと基準画像を再検証する必要がある。
