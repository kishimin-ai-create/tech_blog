# Jestテストの補完が出ない原因は継承された`exclude`だった

## 結論

Jest用のTypeScript設定がアプリ用設定を継承し、テストファイルの`exclude`も引き継いでいた。Jest用設定で`exclude`を空に上書きすると、テストファイルがTypeScriptプロジェクトに含まれ、`expect`などのJestグローバルをエディターが認識できるようになった。

## 発生した問題

`frontend_jest`のテストファイルで、`expect`などJestの名前がエディター補完に出なかった。Jest自体は導入済みで、Jest用設定には`types: ["jest", "node", "vite/client"]`も指定されていた。

## 調査

アプリ用設定はテストを除外していた。

```json
{
  "include": ["src"],
  "exclude": ["src/**/*.test.ts", "src/**/*.test.tsx"]
}
```

Jest用設定はアプリ用設定を継承していたが、`include`だけを上書きし、`exclude`は指定していなかった。

```json
{
  "extends": "./tsconfig.app.json",
  "compilerOptions": {
    "types": ["jest", "node", "vite/client"]
  },
  "include": ["src", "jest.setup.ts"]
}
```

`tsc --showConfig -p tsconfig.jest.json`で継承後の設定を確認すると、`exclude`にテストのglobが残り、`files`にも`.test.tsx`が含まれていなかった。

## 原因

`include`へ`src`を書いても、親設定から継承された`exclude`は解除されない。そのためJest用TypeScriptプロジェクトからテストファイルが除外され、Jestの型情報を読み込む対象にもならなかった。

## 解決方法

Jest用設定で`exclude`を空配列として明示し、親設定の値を上書きした。

```json
{
  "extends": "./tsconfig.app.json",
  "include": ["src", "jest.setup.ts"],
  "exclude": []
}
```

アプリ用設定の除外規則はそのままなので、アプリの型チェックとJestテストの型チェックを別々に保てる。

## 動作確認

```powershell
node .\node_modules\typescript\bin\tsc --showConfig -p tsconfig.jest.json
```

修正後の出力では`exclude`が空で、`src/App.test.tsx`と`src/field.test.tsx`が`files`に含まれた。ビルドとJestテストも成功した。

## 事実・判断・未確認事項

- FACT: 修正前の展開済み設定はテストファイルを除外し、修正後はテストファイルを含んだ。
- INFERENCE: エディター補完が出なかった原因は、Jestの型不足ではなくテストファイルがJest用TypeScriptプロジェクトに属していなかったことだった。
- 未確認: 別のエディターやTypeScript Language Serviceでも同じ補完結果になるかは個別に検証していない。

## まとめ

Jestの型を`types`へ追加しただけでは不十分な場合がある。継承した`tsconfig`の`include`と`exclude`を、展開後の設定と対象ファイル一覧で確認すると、エディターがテストを認識していない原因を切り分けられる。

確認日: 2026年9月28日
