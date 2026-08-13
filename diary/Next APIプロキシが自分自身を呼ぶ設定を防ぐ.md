# Next APIプロキシが自分自身を呼ぶ設定を防ぐ

## はじめに

設定値を起動時に検査すると、デプロイ後に 502 やループのような症状として現れる前に、原因を切り分けやすくなる。

## 対象読者

- Next.js の Route Handler を API プロキシとして使っている人
- フロントエンドとバックエンドを別サービスでデプロイしている人
- `BACKEND_URL` の設定ミスで API が通らない問題を避けたい人

## 問題

フロントエンドの `/api/*` を Next.js 側で受け、サーバー側からバックエンドへ転送する構成では、ブラウザの Network にはフロントエンドと同じドメインの `/api/diaries` が表示される。

これは正常な構成だが、`BACKEND_URL` までフロントエンド自身の origin を指していると、プロキシが自分自身へリクエストを戻してしまう。

## 原因

`frontend/app/api/backend-url.ts` は、実行時の backend origin を `BACKEND_URL` または Render の `BACKEND_HOST` / `BACKEND_PORT` から組み立てる。

以前は `BACKEND_URL` がフロントエンドと同じ origin かどうかを検査していなかった。そのため、デプロイ環境で誤ってフロントエンド URL を設定しても、アプリは起動してしまった。

## 実際にやったこと

`createBackendUrl` にフロントエンド origin を渡せるようにし、`BACKEND_URL` と同じ origin だった場合は設定エラーにするようにした。

対象ファイル:

- `frontend/app/api/backend-url.ts`
- `frontend/app/api/proxy.ts`
- `frontend/next.config.small.test.ts`
- `frontend/app/api/proxy.small.test.ts`

また、`frontend/.env.example` は backend と frontend の URL を明確に分けるプレースホルダーにした。

## OpenAPI URL も明示する

Orval の `OPENAPI_URL` もローカルのデフォルト値に頼らず、必須設定として読むようにした。

対象ファイル:

- `frontend/orval-url.ts`
- `frontend/orval.config.ts`
- `frontend/orval.config.small.test.ts`

これにより、型生成が意図しないローカル URL に依存しにくくなる。

## まとめ

ブラウザから見える `/api/*` がフロントエンドと同じドメインなのは、Next.js の API プロキシ構成では自然な挙動である。問題は、サーバー側の転送先まで同じ origin になることだった。

設定値を起動時に検査すると、デプロイ後に 502 やループのような症状として現れる前に、原因を切り分けやすくなる。
