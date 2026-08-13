# Workerスクリプトを書かずに、`wrangler.jsonc`だけで静的SPAをCloudflare Workersへ配信する

## はじめに

Cloudflare Workersは、サーバーサイドのWorkerスクリプト（`main`）を一切書かずに、静的アセットだけを配信する構成をサポートしている。`wrangler.jsonc`の`assets`ブロックに配信対象ディレクトリと未一致パスの扱いを指定するだけで、Viteのようなビルドツールが出力する純粋な静的SPAをそのままデプロイできる。

## 結論

Cloudflare Workersは、サーバーサイドのWorkerスクリプト（`main`）を一切書かずに、静的アセットだけを配信する構成をサポートしている。`wrangler.jsonc`の`assets`ブロックに配信対象ディレクトリと未一致パスの扱いを指定するだけで、Viteのようなビルドツールが出力する純粋な静的SPAをそのままデプロイできる。

## 対象読者

- Viteなど、ビルド後に静的ファイル一式（`dist/`）を出力するSPAを、Cloudflare Workers上でホスティングしたい人
- `wrangler.toml`/`wrangler.jsonc`の`assets`フィールドの意味を体系的に知りたい人

## 最小構成

pi-labでは、`npm run build`が`dist/`にビルド成果物を出力する。これをそのままWorkerの配信対象にするための`wrangler.jsonc`は次の通りである。

```jsonc
{
	"$schema": "./node_modules/wrangler/config-schema.json",
	"name": "pi-lab",
	"compatibility_date": "2026-08-09",
	"assets": {
		"directory": "./dist",
		"not_found_handling": "single-page-application"
	}
}
```

Cloudflare公式ドキュメント（`developers.cloudflare.com/workers/wrangler/configuration/`）によれば、Workerをデプロイするには最低限`name`・`main`・`compatibility_date`が必要だが、**`main`はassets-onlyのWorker（静的アセットのみを配信し、独自のWorkerスクリプトを持たない構成）では省略可能**と明記されている。上記の設定に`main`が存在しないのはそのためである。

## `assets`フィールドの主要項目

公式ドキュメント（`developers.cloudflare.com/workers/wrangler/configuration/`、`developers.cloudflare.com/workers/static-assets/routing/`）で確認した項目は次の通り。

| フィールド | 役割 | 主な値 |
| --- | --- | --- |
| `directory` | 配信する静的アセットのディレクトリ | 例: `"./dist"` |
| `binding` | Workerスクリプトからアセットを参照する際のバインディング名 | 独自のWorkerスクリプトが無い場合は不要 |
| `not_found_handling` | 静的アセットに一致しないリクエストの扱い（既定: `"none"`） | `"none"` / `"404-page"` / `"single-page-application"` |
| `html_handling` | URLの末尾スラッシュや`.html`拡張子の扱い（既定: `"auto-trailing-slash"`） | `"auto-trailing-slash"` / `"force-trailing-slash"` / `"drop-trailing-slash"` / `"none"` |
| `run_worker_first` | 静的アセットの一致判定より前にWorkerスクリプトを実行するか（既定: `false`） | `boolean`またはルートパターンの配列 |

`not_found_handling`を`"single-page-application"`にすると、静的アセットに一致しないあらゆるリクエストに対して`index.html`を200で返す。これが、`react-router`の`BrowserRouter`（History APIベースのクライアントサイドルーティング）を使うSPAで、`/pi-message`のような深いURLへの直接アクセスやリロードを404にしないために必要な設定である。

## 実装・検証

```text
$ npx wrangler deploy --dry-run
✨ Read 7 files from the assets directory .../dist
Total Upload: 0.36 KiB / gzip: 0.26 KiB
No bindings found.
--dry-run: exiting now.
```

Cloudflareへのログインなしに、`wrangler.jsonc`の構文と`dist`ディレクトリの内容が正しく認識されることを確認できた（`--dry-run`が実際に検証する範囲の限界については[別記事](./20-validate-wrangler-config-with-dry-run.md)で扱う）。

## 適用条件

- バックエンドAPIやSSRを持たない、純粋なクライアントサイドSPA（またはビルド後に静的ファイルへ変換されるサイト）であること。
- Workerスクリプト側でのリクエスト処理（認証、APIプロキシなど）が必要な場合は、`main`を指定し`run_worker_first`で制御対象を絞る構成が別途必要になる（本記事の対象外）。

## 参考資料

- [Cloudflare Workers: Static Assets](https://developers.cloudflare.com/workers/static-assets/)
- [Cloudflare Workers: Static Assets Routing（SPA）](https://developers.cloudflare.com/workers/static-assets/routing/single-page-application/)
- [Cloudflare Workers: Wrangler Configuration（`assets`フィールド）](https://developers.cloudflare.com/workers/wrangler/configuration/)
- 根拠コミット: `64d37e9`
- 確認日: 2026-08-09

## まとめ

Cloudflareへのログインなしに、`wrangler.jsonc`の構文と`dist`ディレクトリの内容が正しく認識されることを確認できた（`--dry-run`が実際に検証する範囲の限界については[別記事](./20-validate-wrangler-config-with-dry-run.md)で扱う）。
