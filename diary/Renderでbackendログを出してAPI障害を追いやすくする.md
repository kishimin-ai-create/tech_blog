# Render で backend ログを出して API 障害を追いやすくする

## 対象読者

- Render 上で backend の挙動を追いたい人
- Hono + Bun の API に最小限の runtime logging を入れたい人
- 障害調査用ログと秘密情報の扱いを分けたい人

## この記事で扱うこと

この記事では、diary backend に runtime logging を追加した変更をまとめます。

扱う範囲は backend の request log、unexpected error log、migration readiness log です。frontend の表示や API 仕様そのものは変更していません。

## 背景

Render 上で API の `500`、`502`、`503` を追っているとき、HTTP status だけでは「backend まで届いているのか」「migration gate で止めているのか」「handler 内で例外が出ているのか」を見分けにくい状態でした。

そこで、backend runtime で最低限のログを出すようにしました。

## 追加したログ

今回追加したログは次の4種類です。

- request 完了ログ
- unexpected request error ログ
- database migration ready / failed ログ
- migration が未 ready のため API を `503` にした warn ログ

request 完了ログでは、次の情報だけを出します。

```text
method
path
status
durationMs
```

request body、Authorization header、DB URL、JWT などは出しません。

## logger を注入できる形にした

`backend/src/shared/logger.ts` に `AppLogger` を追加しました。

production では `consoleLogger` を使い、Render の log collector に拾わせます。テストや dependency injection された app では `noopLogger` を使い、既存テストがログで汚れないようにしています。

```ts
export interface AppLogger {
  error(message: string, meta?: LogMeta): void;
  info(message: string, meta?: LogMeta): void;
  warn(message: string, meta?: LogMeta): void;
}
```

また、unknown な thrown value を安全に扱うため、`errorLogMeta()` で `errorName` と `errorMessage` だけに変換しています。

## Hono app 側のログ

`createApp()` に logger を注入できるようにし、すべての request に対して完了ログを出す middleware を追加しました。

さらに、`ServiceError` ではない unexpected error が起きた場合に、method、path、errorName、errorMessage を出すようにしました。

これにより、handler 内で予期しない DB エラーなどが起きたときも、Render の log から該当 path を追いやすくなります。

## server 側の migration ログ

`createProductionServer()` では、migration の状態変化をログに出します。

- migration が完了したら `database migrations ready`
- migration が失敗したら `database migrations failed`
- `/api/*` が migration 未 ready により止められたら warn

今回のように `/openapi.json` は返るが `/api/diaries` は `503` になるケースでは、warn log から migration gate による `503` だと判断できます。

## テストで確認したこと

`backend/src/app.small.test.ts` では、次を確認しています。

- request 完了時に method / path / status / durationMs がログされる
- unexpected error 時に method / path / errorName / errorMessage がログされる

`backend/src/server.small.test.ts` では、次を確認しています。

- migration 未 ready で API request を `503` にしたとき warn log が出る
- migration failure 時に error log が出る

検証として、次のコマンドが成功しています。

```bash
bun test src/app.small.test.ts
bun test src/server.small.test.ts
bun run typecheck
bun run lint
bun test small.test
bun run test
```

最終的に backend test は 111 tests が通っています。

## 注意点

`consoleLogger` は本番で `console.info` / `console.warn` / `console.error` を使います。そのため lint では `no-console` warning が出ますが、これは Render に runtime events を拾わせるための用途です。

ログには調査に必要な最小限の metadata だけを含め、request body や credential は出さないようにしています。

## まとめ

HTTP status の変化だけを追うと、障害調査はかなり遠回りになります。

今回の変更では、backend が何を受け取り、どこで止め、どの status を返したのかを Render log で追えるようにしました。

特に migration readiness まわりは deploy 時に問題になりやすいので、API request log と migration state log を分けて出せるようにしたのは、次の調査の足場になります。
