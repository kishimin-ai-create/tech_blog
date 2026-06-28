# RenderでNext.jsをStatic Site扱いしてdistがないエラーを避ける

## 対象読者

- Next.js App Router を Render に載せようとしている人
- `Publish directory dist does not exist` でデプロイが止まった人
- フロントエンドとバックエンドを別サービスとして Docker デプロイしたい人

## この記事で扱うこと

この記事では、Next.js フロントエンドが Render 上で Static Site として扱われ、`dist` が見つからずにビルド失敗した問題と、その対応として Docker Web Service の Blueprint に寄せた変更を扱う。

バックエンド API の永続化実装や UI の詳細は扱わない。

## 起きたこと

Render のビルドログでは Next.js のビルド自体は完了していた。

```text
Route (app)
┌ ○ /
├ ○ /_not-found
├ ○ /admin
├ ○ /admin/create
├ ƒ /admin/edit/[id]
├ ƒ /diaries/[id]
└ ○ /login

==> Publish directory dist does not exist!
==> Build failed
```

ポイントは、Next.js の `next build` が `dist` を作る構成ではないことだ。今回の frontend は `frontend/next.config.ts` の rewrites で `/api/*` と `/openapi.json` を backend に中継する。つまり静的ファイルだけを配る構成ではなく、Next.js の Node/Docker サーバとして動かす必要がある。

## 原因

原因は、Render 側が frontend を Static Site として扱い、公開ディレクトリに `dist` を期待していたことだった。

一方、リポジトリ側の frontend は `next start` で動く前提の Next.js アプリであり、`frontend/Dockerfile` も `.next` と `public` を含めて本番起動する構成になっている。

## 対応

`render.yaml` を追加し、frontend/backend をどちらも Docker の Web Service として定義した。

```yaml
services:
  - type: web
    name: diary-backend
    runtime: docker
    dockerfilePath: ./backend/Dockerfile
    dockerContext: ./backend

  - type: web
    name: diary-frontend
    runtime: docker
    dockerfilePath: ./frontend/Dockerfile
    dockerContext: ./frontend
```

これにより、Render が frontend を Static Site として扱うのではなく、`frontend/Dockerfile` を使ってコンテナとして起動する構成になる。

## backend への接続

frontend は API 呼び出しを相対パスで行い、Next.js の rewrites が backend に中継する。

`frontend/next.config.ts` には `BACKEND_URL` を優先しつつ、Render の service reference で渡される `BACKEND_HOST` と `BACKEND_PORT` から内部 URL を作る関数を追加した。

```ts
export function createBackendUrl(
  env: Record<string, string | undefined> = process.env,
): string {
  const backendUrl = env["BACKEND_URL"];
  if (backendUrl) {
    return backendUrl;
  }

  const backendHost = env["BACKEND_HOST"];
  if (!backendHost) {
    return "http://localhost:3000";
  }

  const backendPort = env["BACKEND_PORT"];
  if (!backendPort) {
    return `http://${backendHost}`;
  }

  return `http://${backendHost}:${backendPort}`;
}
```

`frontend/next.config.small.test.ts` では、明示的な `BACKEND_URL` と Render の `BACKEND_HOST` / `BACKEND_PORT` の両方を検証している。

## 注意点

Render の dashboard で手動作成したサービスは、リポジトリの `render.yaml` と自動で一致するとは限らない。Static Site として作った frontend service が残っている場合は、Docker Web Service として作り直すか、Blueprint を反映する必要がある。

また、`frontend/.env.example` にはローカル開発用の URL を置き、実際の `.env` は gitignore で除外した。共有すべきなのは必要なキー名と placeholder であり、環境固有の値ではない。

## まとめ

- Next.js App Router アプリを Render Static Site として扱うと、`dist` 前提で失敗することがある
- 今回の frontend は rewrites を使うため、Docker Web Service として起動する構成が合っている
- `render.yaml` で frontend/backend/database をまとめ、backend への接続は Render の private service network を使う形にした
- `.env.example` は必要な変数の共有に使い、`.env` 本体はコミットしない
