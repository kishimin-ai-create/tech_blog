# OpenAPIのsourceIdでqueryとpathが競合したエラーを読む

## 結論

`Conflicting location for parameter sourceId`は、同じパラメータの位置がqueryとpathで矛盾していることを示していた。値の型だけでなく、Schemaのパラメータメタデータとルートへの配置先を見比べる必要がある。

## 発生した問題

Swagger UIでInternal Server Errorになる相談のあと、次のエラーが提示された。

```json
{
  "message": "Conflicting location for parameter sourceId",
  "data": {
    "key": "in",
    "values": ["query", "path"]
  }
}
```

## 調査

まず`/docs/json`を直接確認するよう案内した。この画面構成ではSwagger UIがOpenAPI定義を読み込むため、定義側の問題とUI側の問題を分けられる。

当初は画像Schemaの`z.file()`の変換も仮説として挙げた。しかし、提示されたエラーの根拠はパラメータ位置の競合であり、`z.file()`を原因とする確認はできていない。

## 原因

位置の違いはURLで表すと分かりやすい。

```text
query: /entities?sourceId=1
path:  /entities/1
```

`.openapi({ param: { in: "path" } })`というメタデータと、ルートの`request.query`を組み合わせれば、位置の情報が一致しなくなる。これは確認ポイントを示す説明例であり、当時の競合コードをそのまま保存したものではない。

## 解決方法

Schema内の`param.in`と、そのSchemaを置く`request.query`または`request.params`を合わせる。現在のcollectorコードは`sourceId`をqueryと宣言し、ルートでもqueryに配置している。

ただし、どの編集だけで元のエラーが解消したかは記録から特定していない。

## 動作確認

2026年10月18日に、現在のアプリへ`app.request("/docs/json")`を実行した。HTTP 200となり、以下の4パスを確認した。

- /health
- /images/{entityId}
- /collectors/sources
- /collectors/entities

Swagger UIをブラウザで表示する確認は未実施である。

## 再発防止

提案として、OpenAPI文書の生成が成功することを回帰テストで確認する。実行時ルートのテストだけでは、文書生成の競合を見逃す可能性がある。

## まとめ

エラーの`key: "in"`は値の形式ではなくパラメータ位置を指す。位置情報を付けたSchemaを再利用するときは、利用先のrequest定義まで確認する。

## 根拠

ユーザー提示のエラー、`app/schemas/collectors/collectors.ts`、`app/backend/src/routes/collectors/collectors.ts`、2026年10月18日のOpenAPI文書取得結果。
