# Honoのc.jsonへPromiseを渡していた型エラーを切り分ける

## 結論

`response.json()`の戻り値はJSON本体ではなく`Promise<any>`だった。`c.json(response.json(), 200)`ではPromiseを渡してしまう。まず`await`し、受信値をSchemaで検証すると、実行時の値と返却型を揃えられる。

## 発生した問題

`app/backend/src/routes/collectors/collectors.ts`で、上流APIから受け取ったJSONを返すハンドラーにTS2345が出た。相談ではanyとSchemaの不整合が疑われていた。

## 調査

`app`直下の`bun run typecheck`で、戻り値に`JSONRespondReturn<Promise<any>, 200>`が含まれることを確認した。期待されていたデータは`{ id: number; name: string }[]`だった。

```ts
return c.json(response.json(), 200);
```

型エラーの直接の原因は、JSON解析結果を待っていないことだった。

## 解決方法

以下は上流の本文が配列そのものである場合の最小例。

```ts
const data: unknown = await response.json();
const sources = SourcesResponseSchema.parse(data);
return c.json(sources, 200);
```

`unknown`は外部データを検証前に信用しないための受け口になる。`parse()`の結果にはSchemaから決まる型が付く。型アサーションでデータの正しさを宣言する必要はない。

実際のcollectorルートは`{ sources: [...] }`という包みを想定しているため、コミット`b570a23`では`data.sources`を検証している。この実装は受信JSON全体をunknownとして検証する形にはなっていない。上の例をそのまま適用するには、包みを含むSchemaを用意する必要がある。

## 動作確認

元のコードでは型チェックの失敗を確認した。ユーザーによる実装修正後は`bun run typecheck`が成功した。ただし実APIのデータ取得とcollectorのハンドラーを通した回帰テストは未確認である。

## 再発防止

提案として、上流の包みを含めてSchemaで検証し、JSON解析と返却データの変換を明示するとよい。Schemaに失敗した場合のHTTPエラー契約も別途必要になる。

## まとめ

Promiseを待つことと、待った値の形を検証することは別の責務である。型エラーを読むときは、まず戻り値にPromiseが残っていないかを確認する。

## 根拠

blog-wordcloudの`app/backend/src/routes/collectors/collectors.ts`、`app/schemas/collectors/collectors.ts`、コミット`b570a23`、調査時のTS2345とその後の型チェック結果。
