# ErrorFallbackの日本語・英語StoryをPlaywright VRTで比較する

## 結論

ErrorFallbackはアプリの実行時例外を意図的に発生させず、Storybookの日本語・英語Storyをiframe URLから開いてVRTする。Page ObjectはStory IDの遷移、見出し、スクリーンショット比較だけを担当し、5つのPlaywrightプロジェクトで各ロケールの基準画像を持つ。

## 背景

実アプリのErrorBoundaryをE2Eで再現すると、例外発生条件に依存する。ErrorFallbackにはすでにロケールごとのStoryがあるため、表示状態を決定的に固定できるStorybookを視覚比較の境界にした。

## 実装

```ts
const openStory = async (storyId: string) => {
  await page.goto(`/iframe.html?id=${storyId}&viewMode=story`);
};

const compareScreenshot = async (name: string) =>
  expect(page).toHaveScreenshot(name, { fullPage: true });
```

日本語と英語は`storyWithLocale("ja")`、`storyWithLocale("en")`で同じ形式に定義し、テスト本体は表示文言を直接持たない。

## 検証結果

`bun run e2e -- e2e/tests/error-fallback.visual.medium.test.ts --update-snapshots`で、日本語・英語×5プロジェクトの10件が成功した。基準画像を更新しない再実行も10件成功した。

## 事実・判断・未確認事項

- FACT: `Google Chrome`、`Microsoft Edge`、`Safari`、`Android (Chrome)`、`iPhone (Safari)`で10件が成功した。
- FACT: Storybook起動とVite previewはPlaywrightのwebServerで分離している。
- INFERENCE: Storyを直接開くことで、実行時例外の再現性に依存せず表示状態を比較できる。
- ASSUMPTION: Storybookのiframe描画が本番アプリのErrorBoundary統合を保証するわけではない。

## 参考

- `frontend/e2e/tests/error-fallback.visual.medium.test.ts`
- `frontend/e2e/pages/error-fallback-page.ts`
- `frontend/src/features/error/views/ErrorFallback.stories.tsx`
- `C:/Users/Kazum/.codex/docs/adr/0015-run-vrt-across-all-playwright-projects.md`

確認日: 2026-09-06
