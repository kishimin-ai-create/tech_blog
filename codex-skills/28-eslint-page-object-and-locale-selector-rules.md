# Page Objectの関数参照とlocale Selector対応表をLintで守る

## 結論

Page Objectの返却オブジェクト内で操作関数を直接定義する構造と、locale Selectorを関数で分岐する構造は、ASTで判定できる範囲をカスタムESLintルールにした。操作関数は返却前に定義し、locale Selectorは`Record<Locale, RegExp>`相当の対応表にする。

## 禁止する形

```ts
return {
  submit: async () => page.getByRole("button").click(),
};

const reloadButtonName = (locale) =>
  locale === "ja" ? /再読み込み/ : /Reload/;
```

## 許可する形

```ts
const submit = async () => page.getByRole("button").click();
return { submit };

const selectors = {
  reloadButton: { ja: /再読み込み/, en: /Reload/ },
};
```

`local/require-e2e-page-object-method-references`は`e2e/pages`のPage Object factoryの返却プロパティを検査する。`local/require-localized-selector-map`は`e2e/selectors`内で`locale`引数を持つ関数を検査する。POMのlocale引数は対象外である。

## 検証結果

`bun run test:eslint`で違反例・許可例を含む30テストが成功し、`bun run typecheck`と`bun run lint`も成功した。

## 制約

Lintは構文上の関数形式を検出できるが、Selectorの文言が適切か、対応表が正しい翻訳かまでは判断しない。その責務は`Record<Locale, ...>`の型と画面テスト、レビューに残る。

確認日: 2026-09-06
