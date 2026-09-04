# StorybookのStoryと振る舞いテストを分離する

## 結論

Storybookはコンポーネントの表示例を残し、ユーザー操作や非同期状態の検証は専用テストへ分離すると、Storyの責務を明確にできます。今回の画像生成フォームでは、Storyから`play`によるinteraction testを削除し、Small／Mediumテストに振る舞い検証を集約しました。

## 背景

画像生成フォームのStoryには、入力、送信、バリデーション、APIエラーを操作で再現する`play`処理が含まれていました。React Hook Formのresolverは状態更新を非同期に行うため、Storyのアサートには待機が必要でした。レビューでは、これは画面描画の遅延ではなく、テスト側が状態更新を待っていない問題として指摘されました。

## 変更した方針

- StoryはDefault、English、入力例、エラー例などの表示バリエーションとして残す。
- `play`、`userEvent`、`waitFor`による操作とアサートはStoryから除去する。
- 振る舞いの検証は、Small／Mediumテストでユーザー操作と結果を直接検証する。

```tsx
export const ValidationError: Story = {
  args: { locale: "ja" },
};
```

操作を含まないStoryは、表示例として読み手が確認できます。入力やエラーの因果関係は、専用テストのテスト名とアサートで表現します。

## 検証

変更後に次のコマンドを実行し、Lint、型検査、Storybookテストが成功しました。

| コマンド | 結果 |
| --- | --- |
| `bun run lint` | pass |
| `bun run typecheck` | pass |
| `bun run test:storybook` | pass（13 files、42 tests） |

## 事実・判断・未確認事項

- FACT: `ImageGenerationForm.stories.tsx`からinteraction testを削除し、各Story定義を維持した。
- FACT: Storybookテストは42件成功した。
- INFERENCE: 非同期resolverの待機責務を専用テストへ移したことで、Storyは表示例に集中できる。
- ASSUMPTION: 実ブラウザでの視覚比較とスクリーンリーダー確認は別途必要である。

## 参考

- `frontend/src/features/image-generation/components/ImageGenerationForm/ImageGenerationForm.stories.tsx`
- `frontend/src/features/image-generation/components/ImageGenerationForm/ImageGenerationForm.small.test.tsx`
- `6d7aec2 test: remove story interaction tests`
- `852e718 test: keep static image generation stories`
