# React Hook Formの専用Hookをどうテストするか

## 結論

専用Hook単体のテストでは、`renderHook`でHookを実行し、React Hook Formの公開APIである`getValues()`と`trigger()`を検証する方法が適切である。DOMを持つフォームコンポーネントのテストでは、同じ値を`toHaveFormValues()`など画面に近い方法で確認する。

## Hookテストのアサート

初期値は全フィールドを一度に比較する。

```ts
const { result } = renderHook(() => useImageGenerationForm());

expect(result.current.getValues()).toEqual({
  text: "",
  foregroundCharacter: "",
  foregroundColor: "#000000",
  backgroundCharacter: "",
  backgroundColor: "#FFFFFF",
  type: imageTypeDefinitions.standard,
});
```

有効性の確認では、`trigger()`の結果を変数へ受けてからアサートすると、検証結果を確認していることが読み取りやすい。

```ts
const isValid = await result.current.trigger();

expect(isValid).toBe(true);
```

## UIテストとの使い分け

Testing Libraryのフォームテストでは、フォームロールに対する`toHaveFormValues()`が案内されている。これはDOMに描画された利用者向けの状態を検証する。一方、HookテストにはDOMがないため、`getValues()`がHookの公開契約に対応する。

## 検証結果

`useImageGenerationForm.small.test.ts`では、初期値、スキーマエラー、妥当な値の検証成功を確認した。対象テストは3件成功し、Smallテスト全体は78件成功した。

## 事実・判断・未確認事項

- FACT: React Hook Formの`getValues()`は現在のフォーム値を取得し、`trigger()`は検証結果をPromiseで返す。
- FACT: Testing LibraryはDOMフォームの値確認に`toHaveFormValues()`を示している。
- INFERENCE: Hook単体とUIコンポーネントでアサート手段を分けると、テスト対象の境界が明確になる。
- ASSUMPTION: 実際の入力要素への初期値反映は、後続の`ImageGenerationForm`統合テストで確認する。

## 参考資料

- [React Hook Form: getValues](https://github.com/react-hook-form/documentation/blob/master/src/content/docs/useform/getvalues.mdx)
- [React Hook Form: defaultValues](https://github.com/react-hook-form/documentation/blob/master/src/content/faqs.mdx)
- [Testing Library: Verify form values](https://github.com/testing-library/testing-library-docs/blob/main/docs/react-testing-library/migrate-from-enzyme.mdx)
