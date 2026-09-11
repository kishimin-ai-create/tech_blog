# 言語切り替えの選択肢をコンポーネントから分離する

## 要約

言語切り替えUIの対応言語をコンポーネント本体へ直書きせず、locale定義と表示用optionsを分離した。言語追加ではデータを追加するだけで済み、表示順も配列の順序として明示できる。

## 背景

`LanguageSwitcher`が内部に言語名や順序を持つと、言語追加のたびにUIロジックを変更する必要がある。さらに、`Object.values`だけに依存すると、テストが表示順を明示しにくい。

## 実装

localeの最小定義と、UIが表示する選択肢を別にした。

```ts
export const localeDefinitions = {
  ja: { label: "日本語" },
  en: { label: "English" },
} as const;

export const languageOptions = [
  { locale: "ja", label: "日本語" },
  { locale: "en", label: "English" },
] as const;
```

コンポーネントはoptionsをmapし、選択されたlocaleを`onChange`へ返すだけにした。テストは固定された値を再利用せず、期待する表示順を文字列配列で表現する。

## 検証

LanguageSwitcherのSmallテストで日本語・英語の順序、キーボード操作、選択イベントを確認した。関連コミットは`ce7ec0f`、`5a12e15`、`bcf8ada`、`c13658b`である。

## 学び

選択肢の追加を「コンポーネントの修正」ではなく「データの追加」に変えるには、値・表示名・順序をUIの入力データとして扱う必要がある。テストも同じデータをimportせず、利用者から見える順序を独立した期待値として持つと回帰を検出できる。
