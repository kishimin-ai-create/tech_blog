# Zodで受信するsnake_caseを検証してcamelCaseへ変換する

## はじめに

外部APIのJSONとフロントエンドへ返したいJSONでは、フィールド名が異なることがある。今回はcollectorの`source_id`と`is_active`を、Honoからは`sourceId`と`isActive`として返したかった。

返したい名前だけでSchemaを定義しても、`parse()`は受信フィールド名を自動で変更しない。受信する形を検証してから変換する流れを試した。

## 前提・環境

Windows、Bun 1.3.13、Zod 4.6.5、@hono/zod-openapi 1.6.3。対象はblog-wordcloudの`EntitiesResponseSchema`。

## やってみた結果

`.object()`に上流の名前を定義し、`.transform()`で公開したい名前へ変更できた。

```ts
const schema = z.object({
  id: z.number().min(1),
  source_id: z.number().min(1),
  name: z.string(),
  is_active: z.boolean(),
}).transform(({ source_id, is_active, id, name }) => ({
  id,
  sourceId: source_id,
  name,
  isActive: is_active,
}));
```

これは要点を抜き出した例で、実装では配列Schemaの要素として使用している。

## 実装・検証

### 受信する名前で検証する

入力は外部APIが実際に送るキーに合わせる。受信データの形が不正なら、変換へ進む前に検証が失敗する。

### 変換後の値を返す

2026年10月18日に、実装済みSchemaへ次の配列を直接渡した。

```json
[{"id":1,"source_id":2,"name":"sample","is_active":true}]
```

観測した結果は次のとおり。

```json
[{"id":1,"sourceId":2,"name":"sample","isActive":true}]
```

## つまずいたところ

受信する名前と返す名前の不一致がparseエラーの相談につながった。受信用と返却用の役割を分けると、検証の対象が明確になった。

## 学んだこと

入力と出力が違うSchemaでは、Schemaの入力型と出力型を同じだと考えないほうがよい。今回確認したのは、実行時に検証・変換した結果である。

OpenAPIにどちらの形が記録されるかは別の確認項目になる。今回の値変換の成功だけで、生成クライアントの型まで正しいとは判断していない。

## まとめ

外部のフィールド名を受け入れる境界で検証し、その後に公開する名前へ変換する。実APIの通信は未確認だが、固定データによる変換結果は確認できた。

## 根拠

`app/schemas/collectors/collectors.ts`、`app/backend/src/routes/collectors/collectors.ts`、コミット`b570a23`。2026年10月18日のBunによるSchema直接実行。
