# pi-lab 技術記事

React、Vite、Vitest、Storybook、Playwrightで構成された「割り切れない研究所」の開発から得た知見をまとめる。各記事は2026年8月6日時点のリポジトリと、Playwright 1.61.1での検証結果に基づく。

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

## 共通の前提

- Node.js 22または24
- Playwright 1.61.1
- Vitest 4.1.10
- Windows 11でのローカル実行と、Ubuntu NobleコンテナでのCI相当検証

バージョン更新後は、各記事のコマンドと基準画像を再検証する必要がある。
