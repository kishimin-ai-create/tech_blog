# 実APIの画像生成E2Eで並列実行とSafariダウンロードを切り分ける

## 結論

全E2Eは45件中41件が成功した。失敗はAPI接続断ではなく、並列実行時のGlyph Forge `429 Too Many Requests`と、Safari系で期待したPlaywrightの`download`イベントが発生しないタイムアウトに分かれた。`--workers=1`で画像生成だけを再実行すると8件成功、Safari系2件が残った。

## 確認した環境

- Mojica API: `http://localhost:5063`
- Glyph Forge: `http://localhost:8080`
- 両コンテナは`docker ps`で稼働を確認した。
- APIログには`POST /images`の`200`と`429`が記録された。

## 検証コマンド

```text
bun run e2e
bun run e2e -- e2e/tests/image-generation.medium.test.ts --workers=1
```

全体では45件中41件成功、画像生成を1 workerで実行した場合は10件中8件成功した。SafariとiPhone Safariでは、Page Objectの`page.waitForEvent("download")`が30秒でタイムアウトした。

## 事実・判断・未確認事項

- FACT: APIコンテナとGlyph Forgeコンテナは起動していた。
- FACT: Glyph Forgeログに並列リクエストへの429があった。
- FACT: Safari系の失敗はダウンロードイベント待ちで発生した。
- INFERENCE: APIのレート制限とブラウザーのダウンロード挙動は、同じ失敗として扱わず分離して調査すべきである。
- ASSUMPTION: Safari系での失敗がアプリのBlob処理、WebKitのダウンロード制約、または応答ヘッダーのいずれに起因するかは未確定である。

## 次の調査

Safari系でレスポンス、`Content-Disposition`、ブラウザーのダウンロード対応を個別に確認する。APIの429対策としては、E2Eの並列度とサービス側レート制限を同じCI条件で管理する必要がある。

確認日: 2026-09-06
