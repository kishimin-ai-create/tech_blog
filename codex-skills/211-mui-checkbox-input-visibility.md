# MUI Checkboxのinputが透明なときにDOMを検証する

## はじめに

MUIのCheckboxをReact Testing Libraryで取得すると、アイコンではなく内部のinputが対象になる。MUI v6.4.12の実装を読み、透明なinputに対する表示検証とヘルパーテキストの紐づけを整理した。

## 前提・環境

- Material UI v6.4.12の既定スロットのソースコード
- React Testing Libraryとjest-domを使う記事のコード例
- ソースの確認であり、この作業ではテストを実行していない

## やってみた結果

`CheckboxRoot` は `SwitchBase` をスタイルしたコンポーネントである。`SwitchBase` の既定inputは `SwitchBaseInput` で、`opacity: 0` が指定される。アイコンはinputとは別に描画される。

```text
Checkbox
└─ CheckboxRoot（styled SwitchBase）
   └─ SwitchBase
      └─ RootSlot
         ├─ InputSlot（SwitchBaseInput: opacity 0）
         └─ アイコン
```

## 実装・検証

### roleで取得する要素

`getByRole("checkbox")` は、`checkbox` ロールを持つ内部の `input type="checkbox"` を取得する。jest-domの `toBeVisible()` は `opacity: 0` を非表示と判定するため、既定のinputに対しては失敗する。

```tsx
expect(screen.getByRole("checkbox", { name: "のんびり" }))
  .toBeInTheDocument();
```

これはDOMへの存在を確認する例であり、アイコンの視覚的な表示を確認するものではない。環境やスロットのカスタマイズによってスタイルは変わり得る。

### ヘルパーテキストの紐づけ

`slotProps.input` に渡したpropsは既定のinputに適用される。`aria-describedby` にヘルパーテキストのIDを指定することで、inputと説明を関連付けられる。

```tsx
<Checkbox slotProps={{ input: { "aria-describedby": "check-description" } }} />
<FormHelperText id="check-description">説明</FormHelperText>
```

## つまずいたところ

見た目のCheckboxとroleで取得するinputを同一視すると、表示されているアイコンと透明なinputの違いを見落とす。実装をたどり、root・input・アイコンの役割を分けて整理した。

## 学んだこと

取得したDOM要素と、画面上の見た目を担当する要素は異なる場合がある。存在、操作、視覚表示のどれを確認したいかで検証方法を選ぶ。

## 参考資料

- [Checkbox v6.4.12実装](https://github.com/mui/material-ui/blob/v6.4.12/packages/mui-material/src/Checkbox/Checkbox.js)
- [SwitchBase v6.4.12実装](https://github.com/mui/material-ui/blob/v6.4.12/packages/mui-material/src/internal/SwitchBase.js)
- [jest-dom toBeVisible](https://github.com/testing-library/jest-dom#tobevisible)

確認日: 2026-11-03
