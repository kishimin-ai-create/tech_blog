# Zodで画像生成フォームのHEXカラー入力を検証する

## 対象読者

フォームで`#RRGGBB`形式の色を受け付けるTypeScriptアプリケーションの開発者。

## スコープ

Mojicaの画像生成スキーマに追加した6桁HEXカラー検証を扱う。色変換、CSS色名、アルファ付きHEXは対象外である。

## 課題

色入力は文字列としてフォームから渡されるため、色として利用できない値も受け取れてしまう。画像生成リクエストの境界で、前景色と背景色を同じ形式に制限する必要があった。

## 実装

Zodの`regex`に、先頭の`#`と16進数6桁を表す正規表現を渡した。

```ts
const hexColor = /^#[0-9A-Fa-f]{6}$/;

const imageGenerationSchema = z.object({
  foregroundColor: z.string().regex(hexColor, {
    message: "foregroundColor.invalid",
  }),
  backgroundColor: z.string().regex(hexColor, {
    message: "backgroundColor.invalid",
  }),
});
```

エラー文はスキーマへ表示言語で直書きせず、`foregroundColor.invalid`と`backgroundColor.invalid`というメッセージキーを返す。フォーム側で現在のロケールに対応する日本語・英語のメッセージへ変換するため、検証ルールと表示文言を分離できる。

## テスト

スキーマのSmallテストでは、前景色と背景色それぞれについて不正なHEX値を渡し、該当する`path`とメッセージキーを持つIssueを確認する。フォームのStorybookでは実際に不正な色を入力したときのアクセシブルなエラー表示も確認できる。

## 検証

アプリリポジトリでは`bun run test`、`bun run typecheck`、`bun run lint`を実行し、いずれも成功した。テスト結果は21ファイル、103テスト成功だった。

## 事実・判断・未確認事項

- FACT: `foregroundColor`と`backgroundColor`へ`^#[0-9A-Fa-f]{6}$`の検証を追加した。
- FACT: 検証エラーはロケール非依存のメッセージキーで返している。
- INFERENCE: スキーマで形式を検証し、UIで翻訳する分離により、同じルールを別の表示言語でも利用できる。
- 未確認事項: アルファ付きHEXやCSSの色名を受け付ける要件は確認していない。

## まとめ

フォームの色入力を安全に扱うには、APIへ渡す前のスキーマ境界で許可形式を明示する。正規表現とメッセージキーを組み合わせると、検証ロジックとi18nを別の責務として保てる。

## 参考

- `frontend/src/features/image-generation/schemas/imageGenerationSchema.ts`
- `frontend/src/features/image-generation/schemas/imageGenerationSchema.small.test.ts`
- `6388f2b feat: validate image generation hex colors`
- `9f8d5d9 test: cover color validation feedback`
