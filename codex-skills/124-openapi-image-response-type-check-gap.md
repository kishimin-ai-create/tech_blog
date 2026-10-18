# Honoで画像とJSONを返すルートの型検査が不整合を見逃した理由

## 結論

今回インストールされていた@hono/zod-openapi 1.6.3では、`image/png`の返却型が汎用の`Response`になった。この型を含むルートでは、500のJSONがSchemaと異なっていても、`Response`として適合し得る。

## 発生した問題

画像ルートの500レスポンスは、Schemaが`{ message: string }`なのに実装は`{ error: string }`だった。それでも型エラーにならないという疑問が出た。

## 調査

インストール済みの`@hono/zod-openapi/dist/index.d.mts`にある`ReturnJsonOrTextOrResponse`を確認した。

| Content-Type | 型の扱い |
| --- | --- |
| application/json | Schemaに対応するTypedResponse |
| text/plain | TypedResponse |
| image/png | Response |

Hono側の`c.json()`の戻り値も、`Response & TypedResponse<...>`という交差型だった。

## 原因

このルートの戻り値の許容範囲は、概念的には次のようになる。

```ts
Response | TypedResponse<{ message: string }, 500, "json">
```

`c.json({ error: "..." }, 500)`は後者とは一致しないが、前者の`Response`としては適合する。JSONだけのcollectorルートとの差はここにあった。

これは調査したローカル依存の型定義に基づく説明であり、すべてのバージョンに一般化していない。

## 解決方法

返却キーをSchemaへ合わせる必要がある。以下は提案であり、今回のコミットではcatchの不整合は残っている。

```ts
return c.json({ message: "外部APIの取得に失敗しました" }, 500);
```

Schemaを定義しただけで、返すオブジェクトが自動で改名・検証されるわけではない。

## 動作確認

型定義の確認と、現状コードに対する`bun run typecheck`の成功を確認した。現状で型チェックが通ることは、返却JSONの契約が正しい証明にはならない。

## 再発防止

提案として、画像成功時とは別に500時のステータス・Content-Type・JSONキーをテストする。型の広がりをテストで補う方針であり、この回帰テストはまだ追加していない。

## まとめ

型エラーが出ない理由を調べるときは、Schemaから生成される型だけでなく、unionに広い型が含まれていないかも確認する。

## 根拠

`app/backend/src/routes/images/images.ts`、ローカルの@hono/zod-openapiとHonoの型定義、2026年10月18日の型チェック結果。
