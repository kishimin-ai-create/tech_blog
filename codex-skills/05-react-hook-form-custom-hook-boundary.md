# React Hook Formを機能専用Hookへ切り出す境界を決める

## 結論

`useForm`は常にカスタムHookへ切り出す必要はない。しかし、Zod resolver、初期値、フォーム型を機能固有の契約としてまとめる場合は、`useImageGenerationForm`のような専用Hookに分けると責務をフォームUIから分離できる。

## 背景

画像生成フォームでは、React Hook Formの状態管理とZodスキーマの接続をフォームコンポーネントが直接持つ案も考えられる。一方、設計書は`useImageGenerationForm.ts`の責務を、フォーム状態、resolver接続、初期値に限定している。

## 実装

```ts
const defaultValues = {
  text: "",
  foregroundCharacter: "",
  foregroundColor: "#000000",
  backgroundCharacter: "",
  backgroundColor: "#FFFFFF",
  type: imageTypeDefinitions.standard,
} satisfies DefaultValues<ImageGenerationFormValues>;

export const useImageGenerationForm = () =>
  useForm<ImageGenerationFormValues>({
    defaultValues,
    resolver: zodResolver(imageGenerationSchema),
  });
```

`type`の値はUI側で別の文字列Unionを再定義せず、共有する実行時定義から取得する。初期入力は空にし、色だけは生成画面の中立値として黒と白を設定した。

## 判断基準

- `useForm()`の呼び出し以外に機能固有の設定がないなら、フォーム内に直接書いてもよい。
- resolver、初期値、型、共通の状態変換がまとまるなら、専用Hookへ切り出す価値がある。
- Hookは送信APIや画面表示を所有せず、フォーム状態の境界に留める。

## 検証

`useImageGenerationForm.small.test.ts`で、全初期値、無効な`text`のエラー、妥当な値の検証成功を確認した。`bun run test:small`は17ファイル・78テスト成功、PRカバレッジはStatements 97.48%、Branches 94.91%、Functions 95.55%、Lines 97.38%だった。

## 事実・判断・未確認事項

- FACT: React Hook Form公式は`useForm({ defaultValues })`で初期値を設定する例を示している。
- FACT: `ImageGenerationForm.md`はフォームHookにresolver接続と初期値を割り当てている。
- INFERENCE: この機能ではHook分離が責務境界に合う。
- ASSUMPTION: 黒と白の色初期値は現在のUI判断であり、将来のデザイン仕様で変更される可能性がある。

## 参考資料

- [React Hook Form: Initialize form values with defaultValues](https://github.com/react-hook-form/documentation/blob/master/src/content/faqs.mdx)
- `docs/v1/ui/components/ImageGenerationForm.md`
- `frontend/src/features/image-generation/hooks/useImageGenerationForm.ts`
