# ImageGenerationFormの状態をStorybookで再現可能にする

## 対象読者

フォームの通常状態、入力エラー、送信中、APIエラーをStorybookで確認したいReact開発者。

## スコープ

Mojicaの`ImageGenerationForm`で追加したStorybook状態とMSW設定を扱う。Storybookの導入手順やVisual Regressionの運用は扱わない。

## 課題

フォームの表示確認だけでは、入力エラーやAPI応答後の状態を再現しにくい。そこで、状態ごとにStoryを分け、API境界はMSWのハンドラーで固定した。

## 実装

`ImageGenerationForm.stories.tsx`には、Default、English、Filled、ValidationError、Submitting、Success、文字数超過、各APIエラーのStoryを定義した。APIエラーはレスポンスコード、HTTPステータス、メッセージをMSWで設定する。

```tsx
export const Submitting: Story = {
  args: { locale: "ja" },
  parameters: {
    msw: {
      handlers: [
        getPostImagesMockHandler(() => new Promise<ArrayBuffer>(() => {})),
      ],
    },
  },
  play: async ({ canvasElement }) => {
    await fillRequiredFields(canvasElement);
    await submitForm(canvasElement);
    await expect(
      within(canvasElement).getByRole("button", { name: "生成中..." }),
    ).toBeDisabled();
  },
};
```

送信中StoryではPromiseを解決しないハンドラーを使い、画面を送信中に留める。エラーStoryでは実際の`POST /images`をMSWで失敗させ、フォームに表示されるアラートを確認する。入力操作と確認にはStorybookのテストユーティリティを使う。

## 文字数超過の状態

入力欄へ64文字を超える文字列、または128文字を超える文字を入力するStoryも追加した。これにより、エラーメッセージとアクセシブルなエラー関連付けをブラウザ上で確認できる。

## 検証

アプリリポジトリで`bun run build-storybook`を実行し、Storybookのビルドが成功した。通常のテストは21ファイル、103テストが成功した。

## 事実・判断・未確認事項

- FACT: Storybookに入力、送信中、成功、APIエラー、文字数超過の状態を追加した。
- FACT: API応答の再現には生成済みMSWハンドラーと`http.post`を使用している。
- INFERENCE: 状態ごとのStoryを固定データで保持すると、手動確認時の再現条件が明確になる。
- 未確認事項: Storybookの全Storyを実ブラウザで巡回するVisual Regressionは今回実施していない。

## まとめ

フォームのStorybookは初期表示だけでなく、ユーザー操作後に到達する状態を独立したStoryとして定義すると確認しやすい。ネットワーク境界をMSWで固定すれば、送信中やAPIエラーも再現可能な表示状態として扱える。

## 参考

- `frontend/src/features/image-generation/components/ImageGenerationForm/ImageGenerationForm.stories.tsx`
- `frontend/src/api/endpoints/image/image.msw.ts`
- `6276166 feat(storybook): add image generation form states`
- `61f4098 test: cover oversized form input story`
