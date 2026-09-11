# Reactテストの`userEvent`と`render`を共通setupへ集約する

## 要約

React Testing Libraryの`render`と`userEvent.setup()`を各テストで個別に繰り返す代わりに、共通の`setup`関数へ集約した。テストごとのArrangeを短くしつつ、返り値の型とTesting Libraryのcleanupを維持できる。

## 実装

共通setupは要素を受け取り、userとrender結果をまとめて返す。

```tsx
export const setup = (
  element: ReactElement,
  options?: Parameters<typeof testingLibraryRender>[1],
) => ({
  user: userEvent.setup(),
  ...testingLibraryRender(element, options),
});
```

i18nが必要なテストには、Providerを含める専用setupを追加し、各テストがProviderの構成を手書きしないようにした。

## 使い分け

ユーザー操作があるテストでは`const { user } = setup(...)`を使う。表示だけを確認するテストでは通常の`render`を使ってもよく、すべてを無理に共通setupへ置き換えない。テストのWhatが読みやすいことを優先する。

## 検証

AppHeader、AppFooter、ColorPickerFieldなどのSmallテストで共通setupを利用し、`bun run test:small`の49テストが成功した。関連コミットは`5aac1fb`、`56f877a`、`6b593c8`である。

## 学び

共通化の目的は行数削減ではなく、テスト環境の初期化方法を統一することにある。ユーザー操作とProviderの準備を同じ境界で管理すると、テストごとの微妙な差を減らせる。
