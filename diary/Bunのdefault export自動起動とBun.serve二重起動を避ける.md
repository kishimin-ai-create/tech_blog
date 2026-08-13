# Bunのdefault export自動起動とBun.serve二重起動を避ける

## はじめに

`backend/src/index.ts` が次の 2 つを同時に行っていたことが原因だった。

## 対象読者

- Bun で Hono アプリを本番起動している人
- `EADDRINUSE` が出ているのに、明示的に同じプロセスを二重起動した覚えがない人
- Render のコンテナ上で Bun backend を起動している人

## エラー

backend コンテナの起動時に、次のエラーが出た。

```text
error: Failed to start server. Is port 10000 in use?
code: "EADDRINUSE"
```

ログには Bun 側の entrypoint 処理で `Bun.serve(entryNamespace.default)` が呼ばれている様子が出ていた。

## 原因

`backend/src/index.ts` が次の 2 つを同時に行っていたことが原因だった。

- Hono app を default export する
- `import.meta.main` の中で `Bun.serve(...)` を手動実行する

Bun は entrypoint の default export を server config として扱って自動起動できる。そこに手動の `Bun.serve` が加わると、同じ port に対して 2 回 listen しようとして `EADDRINUSE` になる。

## 実際にやったこと

`backend/src/index.ts` から手動 `Bun.serve` を削除し、default export を Bun が読む server config にした。

```ts
const config = createRuntimeConfig();
export const app = createProductionApp();

export default {
  fetch: app.fetch,
  port: config.port,
};
```

Hono app は `export const app` として残した。これにより、テストからは `module.app.request(...)` でアプリを直接検証できる。

## テスト

`backend/tests/integrations/production-entrypoint.medium.test.ts` に、default export が Bun server config であることを確認するテストを追加した。

```ts
expect(module.default).toMatchObject({
  fetch: expect.any(Function),
  port: 10000,
});
```

これにより、entrypoint が「Bun に読ませる server config」を export していることをテストで固定できる。

## 検証

修正後に backend で次の確認を行った。

```bash
bun test tests/integrations/production-entrypoint.medium.test.ts
bun run typecheck
bun run lint
bun run test
```

全体テストでは 95 tests が通った。

## 注意点

Bun の起動方式は、Node.js の一般的な `server.listen(...)` の感覚だけで見ると見落としやすい。entrypoint の default export を Bun が特別に扱う場合、手動 `Bun.serve` と併用しないほうがよい。

今回の修正では、Hono app 自体のルーティングや API の挙動は変えていない。変更範囲は production entrypoint の起動方式に限定している。

## まとめ

- `EADDRINUSE` の原因は、Bun の default export 自動起動と手動 `Bun.serve` の二重 listen だった
- default export を Bun server config に寄せ、手動 `Bun.serve` を削除した
- Hono app は named export として残し、テスト可能性を維持した
- entrypoint の起動契約は integration test で固定した
