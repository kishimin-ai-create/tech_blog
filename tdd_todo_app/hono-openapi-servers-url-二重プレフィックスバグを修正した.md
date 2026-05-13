# hono-openapi の `servers[].url` と `describeRoute` パスが二重になるバグを修正した

## エラー概要

`hono-openapi` で OpenAPI ドキュメントを生成したとき、Swagger UI の "Try it out" や生成 SDK が
正しい URL を使わず、`/api/v1/api/v1/apps` のような二重プレフィックスのパスにリクエストを送ってしまう問題が発生していた。

---

## 原因

`hono-openapi` の `openAPIRouteHandler` は、生成する OpenAPI spec の各 path を
**`servers[].url` + `describeRoute` で登録したパス文字列** として解決する。

問題のあった設定は次のとおり。

```ts
// backend/src/infrastructure/hono-app.ts（修正前）
app.get(
  '/doc',
  openAPIRouteHandler(app, {
    documentation: {
      info: { title: 'TDD Todo App API', version: '1.0.0', ... },
      servers: [{ url: '/api/v1', description: 'API v1' }],  // ← ここが問題
    },
  }),
);
```

一方、`describeRoute` を使って登録したルートのパスはすべて `/api/v1/...` から始まっていた。

```ts
app.post('/api/v1/apps', describeRoute({ ... }), handler);
app.get('/api/v1/apps/:appId', describeRoute({ ... }), handler);
// ...
```

OpenAPI spec の仕様では、クライアントが実際にリクエストを送る URL は次のように合成される。

```
実際の URL = servers[0].url + paths のキー
           = '/api/v1'     + '/api/v1/apps'
           = '/api/v1/api/v1/apps'   ← 二重になる
```

Hono では `app.post('/api/v1/apps', ...)` と登録した場合、パスは `/api/v1/apps` のまま
spec に書き出されるため、`servers.url` 側に `/api/v1` を設定すると必ず二重になる。

---

## 修正

`servers[].url` を `/api/v1` から `/` に変更する。
ルートパスにプレフィックスが含まれているので、server URL 側は root だけ示せば十分。

```ts
// backend/src/infrastructure/hono-app.ts（修正後）
servers: [{ url: '/', description: 'API root' }],
```

これで合成結果は次のようになる。

```
実際の URL = '/'  + '/api/v1/apps'
           = '/api/v1/apps'   ← 正しい
```

---

## ポイント：`servers.url` の役割を正しく理解する

`servers[].url` は「このパス以下に API がある」というベースを示すものではなく、
OpenAPI クライアントが path を付け足す **起点の URL** を示す。

| パターン | servers.url | path | 合成結果 |
|---|---|---|---|
| ✅ 正しい | `/` | `/api/v1/apps` | `/api/v1/apps` |
| ❌ 二重になる | `/api/v1` | `/api/v1/apps` | `/api/v1/api/v1/apps` |
| ✅ 別の正しい形 | `/api/v1` | `/apps` | `/api/v1/apps` |

Hono でルートを登録するときにフルパス（`/api/v1/apps`）を書いている場合は、
`servers.url` は `'/'` にしておくのが最も混乱が少ない。

逆に、`servers.url` を `'/api/v1'` にしたいなら、Hono のルート登録を
`app.basePath('/api/v1')` などを使ってプレフィックスを除いた形（`/apps`）で登録する必要がある。

---

## まとめ

- `hono-openapi` は `servers[].url + path` を連結して最終 URL を生成する
- Hono でルートをフルパス（`/api/v1/...`）で登録しているなら `servers.url` は `'/'` にする
- Swagger UI の "Try it out" が 404 や不正な URL になるときは、まず `servers.url` と実際のルートパスの組み合わせを確認する
