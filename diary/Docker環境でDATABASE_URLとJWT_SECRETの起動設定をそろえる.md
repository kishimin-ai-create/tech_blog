# Docker環境でDATABASE_URLとJWT_SECRETの起動設定をそろえる

## はじめに

原因は 2 つあった。

## 対象読者

- Hono + Bun の backend を Docker で動かしている人
- ローカルでは動くのにコンテナ上で環境変数不足になる問題を調べている人
- `.env.example` と実行時設定のずれを減らしたい人

## 背景

backend の Docker イメージを作ったあと、起動時に次のようなエラーが出た。

```text
error: DATABASE_URL is required.
```

その後、`DATABASE_URL` まわりを直すと次は次のエラーになった。

```text
error: JWT_SECRET is required.
```

どちらも `backend/src/infrastructures/config.ts` の `createRuntimeConfig` が起動時に必須環境変数を検証しているため発生していた。

## 原因

原因は 2 つあった。

1. runtime config が `DATABASE_URL` 直指定だけを見ていた
2. `backend/.env.example` に `JWT_SECRET` が載っていなかった

Docker イメージでは `.env` を含めない。これは正しい。秘密情報や環境固有の値をイメージに焼き込まないためだ。

一方で、実行環境から `DB_HOST` / `DB_NAME` / `DB_USER` / `DB_PASSWORD` のような分割された DB 設定が渡される場合、backend がそれを DSN に変換できないと起動できない。

## 実際にやったこと

`createRuntimeConfig` を、次の順序で DB 接続設定を解決する形にした。

1. `DATABASE_URL` があればそれを使う
2. なければ `DB_HOST` / `DB_NAME` / `DB_USER` / `DB_PASSWORD` / `DB_PORT` から PostgreSQL URL を組み立てる
3. どちらもなければ従来どおり `DATABASE_URL is required.` を出す

この挙動は `backend/src/infrastructures/config.small.test.ts` に追加した。

```ts
expect(config.databaseUrl).toBe(
  "postgresql://diary_user:secret%20password@database.example:5433/diary_db",
);
```

また、`backend/.env.example` に `JWT_SECRET` の placeholder を追加した。

```env
JWT_SECRET=change_me_to_a_long_random_secret
```

実際の `backend/.env` にもローカル開発用の値を追加したが、`.env` 本体は gitignore の対象なのでコミットしていない。

## Render 側の設定

`render.yaml` では backend service に次の環境変数を渡す。

```yaml
- key: DATABASE_URL
  fromDatabase:
    name: diary-db
    property: connectionString
- key: JWT_SECRET
  generateValue: true
```

Blueprint から作成される場合、Render が `JWT_SECRET` を生成し、PostgreSQL の connection string を `DATABASE_URL` として渡す想定になっている。

手動作成した service で同じエラーが出る場合は、Render dashboard 側に `JWT_SECRET` が設定されているかを確認する必要がある。

## 注意点

`.env.example` に書くのは placeholder までにする。実際のシークレット、DB パスワード、接続文字列は `.env` や Render の Environment Variables に置く。

また、エラーを避けるために `.env` を Docker イメージへコピーするのは避けた。デプロイ先ごとの値は runtime environment で渡すほうが安全で、コンテナイメージの再利用もしやすい。

## まとめ

- Docker イメージには `.env` を含めない
- runtime config は `DATABASE_URL` と DB 分割変数の両方を受けられるようにした
- `JWT_SECRET` は必須なので `.env.example` に placeholder を追加した
- Render では `DATABASE_URL` と `JWT_SECRET` が backend service に入っているかを確認する
