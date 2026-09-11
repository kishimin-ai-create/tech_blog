# React UIの型を生成APIモデルから分離する

## 要約

APIクライアントが生成した型をReact UIへ直接持ち込まず、UIが必要とする型を共有層で定義し、API境界で変換する設計をMojicaへ適用した。これにより、API仕様の変更と表示コンポーネントの責務を切り分けられる。

## なぜ分離するのか

`ImageTypeSelect`は画像種別を選択するUIだが、選択肢のラベルや順序は表示側の責務である。一方、APIのenumはサーバー契約であり、Orvalなどの生成結果に依存する。UIコンポーネントが生成モデルをimportすると、API再生成による型変更がUIの内部実装へ直接波及する。

## 実装

UI側に`ImageType`を定義し、選択肢はUI用の値とラベルを持つ配列として管理した。

```ts
export type ImageType = "standard" | "x-background" | "x-icon";

export const imageTypeOptions = [
  { value: "standard" },
  { value: "x-background" },
  { value: "x-icon" },
] as const;
```

featureのUIから生成APIモデルへのimportはLintで禁止し、リクエスト生成などの境界で必要な変換を行う方針にした。

## 検証

`bun run lint`と`bun run typecheck`が成功し、ImageTypeSelectのSmallテストとStorybookテストも成功した。関連コミットは`5d3c997`、`0ead52d`、`33c5e69`である。

## 学び

APIの型をそのまま使うことは短期的には簡単だが、UI固有の表示順・ラベル・選択肢追加までAPI契約へ結び付けてしまう。UIが所有する型とAPI境界の変換を明示すると、双方の変更理由を独立して説明できる。

## 参考

- `frontend/src/types/image-type.ts`
- `frontend/src/features/image-generation/components/ImageTypeSelect/image-type-options.ts`
- `frontend/eslint.config.mjs`
