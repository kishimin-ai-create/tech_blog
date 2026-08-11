# `wrangler deploy --dry-run`はログイン不要で試せるが、サーバー側の検証は再現しない

## 結論

`npx wrangler deploy --dry-run`は、Cloudflareへのログインや実際のAPI呼び出しなしに、`wrangler.jsonc`の構文と配信対象アセットの内容をローカルで確認できる。ただし、これはあくまでローカルでの事前確認であり、Cloudflareのサーバー側（API）が行う検証（`_redirects`の構文チェックなど）までは再現しない。ローカルで問題が無くても、実際のデプロイでサーバー側の検証に引っかかることがある。

## 実際に確認できたこと

pi-labで`wrangler.jsonc`を追加した際、次のコマンドをログインなしで実行できた。

```text
$ npx wrangler deploy --dry-run

 ⛅️ wrangler 4.120.0
────────────────────
✨ Read 7 files from the assets directory .../dist
Total Upload: 0.36 KiB / gzip: 0.26 KiB
No bindings found.
--dry-run: exiting now.
```

このコマンドは、`assets.directory`に指定したディレクトリから何ファイル読み込めたか、`wrangler.jsonc`のJSON構文が正しいか、といったローカルで完結する内容を検証してくれる。Cloudflareアカウントへのログイン（`wrangler login`）は不要だった。

## 実際のデプロイで初めて見つかったエラー

一方、実際にCloudflareダッシュボードから「保存してデプロイ」を実行したところ、`--dry-run`では検出されなかった次のエラーで失敗した。

```text
✘ [ERROR] A request to the Cloudflare API (/accounts/.../workers/scripts/pi-lab/versions) failed.

Invalid _redirects configuration:
Line 1: Infinite loop detected in this rule. This would cause a redirect to strip `.html` or `/index` and end up triggering this rule again. [code: 100324]
```

エラーメッセージにある通り、これはCloudflareのAPI（`/accounts/.../workers/scripts/pi-lab/versions`）へのリクエストが失敗したことによるものである。`--dry-run`の出力ログには`"--dry-run: exiting now."`とあり、このコマンドは実際にAPIへリクエストを送る前の段階で処理を終了している。つまり、`_redirects`の構文検証のようなサーバー側の処理は、`--dry-run`では実行されないため、ローカルでは検出できなかった。

## 学び

- `--dry-run`は「設定ファイルの構文が正しいか」「意図したファイルが対象になっているか」を、ログイン・API呼び出しなしに素早く確認するのに向いている。
- 一方で、「Cloudflare側が実際に受け入れる内容かどうか」（`_redirects`の妥当性、アカウント固有の制限など）は、`--dry-run`では検証されない。この種の検証はサーバー側で行われるため、実際にデプロイして初めて分かる。
- 「ローカルの検証が通ったから大丈夫」と判断せず、ローカル検証とサーバー側検証は別物として扱う。今回のケースでは、`--dry-run`が成功した後の本番デプロイで初めて`_redirects`の問題が発覚した。

## 未確認事項

`--dry-run`が具体的にどこまでの範囲を検証し、どこから検証しないのかは、公式ドキュメントに包括的な一覧があるわけではなく、今回の1つの事例（`_redirects`検証を再現しなかったこと）から観察した内容に基づく。他の設定項目（`html_handling`の値の妥当性など）が`--dry-run`で検証されるかどうかは未確認である。

## 参考資料

- [Wrangler Commands: `deploy`](https://developers.cloudflare.com/workers/wrangler/commands/workers/#deploy)
- [Cloudflare Workers: Static Assets `_redirects`](https://developers.cloudflare.com/workers/static-assets/redirects/)
- 根拠コミット: `64d37e9`
- 確認日: 2026-08-09
