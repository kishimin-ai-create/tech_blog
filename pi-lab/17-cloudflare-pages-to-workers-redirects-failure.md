# Cloudflareの「Pages」のつもりで用意した`_redirects`が、実際にはWorkers Buildsでデプロイ失敗の原因になった話

## 結論

Cloudflareは現在、Pagesの機能をWorkersへ統合する移行を進めており、ダッシュボードで新規プロジェクトを作成すると、見た目には「Pagesを作る」つもりでも実際には**Workers Builds**（`wrangler deploy`によるGit連携デプロイ）に案内されることがある。この2つは`_redirects`ファイルの扱いが異なり、classic Pagesの定番であるSPAキャッチオールルール（`/* /index.html 200`）は、Workers static assetsの`_redirects`バリデーターに**無限ループ**として拒否される。SPAフォールバックは`wrangler.jsonc`の`assets.not_found_handling`で設定する必要がある。

## 影響

pi-labの初回Cloudflareデプロイが失敗し、サイトを公開できなかった。

## 時系列

1. 「Cloudflare Pagesへデプロイしたい」という要望を受け、classic Pages（GitHub連携、静的ホスティング）を前提に、SPAフォールバック用の`public/_redirects`（`/* /index.html 200`）、セキュリティヘッダー用の`public/_headers`、Node版固定用の`.nvmrc`を追加してマージした。
2. Cloudflareダッシュボードで「Create application」→「Continue with GitHub」→対象リポジトリを選択したところ、セットアップ画面には次のように表示された。

   > アプリケーションをセットアップする
   > Worker プロジェクトを構成し、Cloudflare にデプロイします。

   この文言から、想定していたclassic Pagesではなく、**Workers Builds**（Workersのネイティブなgit連携機能）に案内されていることが判明した。
3. デプロイコマンドの既定値を確認したところ、本番ブランチは`npx wrangler deploy`、非本番ブランチ（詳細設定内）は`npx wrangler versions upload`で、これらはCloudflare公式ドキュメントの既定値と一致していた。
4. 「保存してデプロイ」を実行したところ、ビルドが失敗した。

   ```text
   ✘ [ERROR] A request to the Cloudflare API (.../workers/scripts/pi-lab/versions) failed.

   Invalid _redirects configuration:
   Line 1: Infinite loop detected in this rule. This would cause a redirect to strip `.html` or `/index` and
   end up triggering this rule again. [code: 100324]
   ```

5. 原因調査の結果、Workers static assetsでは`_redirects`/`_headers`はclassic Pagesと同じ構文でサポートされているが、SPAフォールバック（未一致パスに対して`index.html`を200で返す動作）は`_redirects`のキャッチオールルールではなく、`wrangler.jsonc`の`assets.not_found_handling`という別の設定項目で行う仕様だと分かった。
6. `wrangler.jsonc`を追加し、`public/_redirects`を削除する修正を行い、マージした。

## 原因

### 確定している原因

Workers static assetsの`_redirects`バリデーターが、classic Pagesの定番であるSPAキャッチオールルール`/* /index.html 200`を「`.html`または`/index`を剥がすリダイレクトが同じルールに再度マッチし、無限ループになり得る」として拒否する。これはビルドログのエラーメッセージから直接確認できる。

### 仮説（未確認）

Cloudflareのダッシュボード再編（「Workers & Pages」という単独メニューが無くなり、「ビルド」「コンピュート」等のカテゴリに再編された）が、公式ドキュメントの更新より先行しているように見えた。ドキュメント自体は現在も「Workers & Pagesページへ行く」という案内のままで、新しいカテゴリ名（ビルド／コンピュート等）に言及していない。このドキュメントとUIのギャップが、「Pagesのつもりで進めたら実はWorkersだった」という混乱の一因になった可能性がある。これはCloudflare公式ドキュメントには明記されておらず、あくまで観察に基づく推測である。

## 対応

`wrangler.jsonc`を新規作成した。

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

`public/_redirects`は削除した。`public/_headers`は、Workers static assetsでも同一構文でサポートされることを公式ドキュメント（`developers.cloudflare.com/workers/static-assets/headers/`）で確認したうえで、変更せず維持した。

## 検証結果

```text
$ npx wrangler deploy --dry-run
✨ Read 7 files from the assets directory .../dist
Total Upload: 0.36 KiB / gzip: 0.26 KiB
No bindings found.
--dry-run: exiting now.
```

Cloudflareへのログインなしで、`wrangler.jsonc`の構文と`dist`の内容が正しく認識されることを確認した。`npm run build`・`npx tsc --noEmit`・`npx eslint .`もすべて成功した。

Cloudflareダッシュボード上での実際の再デプロイ結果は、本記事の執筆時点では未確認である。

## 学び

- Cloudflareの「Pages」と「Workers（static assets）」は、どちらもGit連携で静的サイトをホスティングできるが、設定ファイルの仕組みが異なる。ダッシュボードの文言（「Pages プロジェクト」か「Worker プロジェクト」か）を作業前に必ず確認する。
- `_redirects`のSPAキャッチオールルールは、classic Pagesでは機能するが、Workers static assetsでは仕様として拒否される。同じ拡張子・同じ場所のファイルでも、配信基盤が変われば挙動が変わり得るという前提を持つ。
- エラーメッセージ自体が原因調査の最も確実な入り口になる。今回は`Invalid _redirects configuration`というメッセージから、公式ドキュメントで正しい設定方法（`not_found_handling`）へ辿り着けた。
- Cloudflareのようにダッシュボードの再編が速いサービスでは、公式ドキュメントの記述が実際のUIより古いことがあり得る。「ドキュメント通りの画面が見当たらない」こと自体を異常と決めつけず、実際に表示されている文言を優先して判断する。

## 制約・未確認事項

- Cloudflareダッシュボード上での実際の再デプロイ結果（成功したか、公開URLで`/pi-message`への直接アクセスが正しく`index.html`にフォールバックするか）は未確認。
- ダッシュボードの「詳細設定」内に表示された`npx wrangler versions upload`が、実際に非本番ブランチ専用のフィールドであることは、画面テキストの並び順からの推測であり、スクリーンショット等で直接確認したものではない。
- ダッシュボードのカテゴリ再編（「ビルド」「コンピュート」等）と、Pages機能のWorkersへの統合の因果関係は仮説であり、Cloudflare公式の説明として確認したものではない。

## 参考資料

- [Cloudflare Workers: Static Assets](https://developers.cloudflare.com/workers/static-assets/)
- [Cloudflare Workers: Static Assets Routing（`not_found_handling`）](https://developers.cloudflare.com/workers/static-assets/routing/single-page-application/)
- [Cloudflare Workers: Static Assets `_headers`](https://developers.cloudflare.com/workers/static-assets/headers/)
- [Cloudflare Workers: Static Assets `_redirects`](https://developers.cloudflare.com/workers/static-assets/redirects/)
- [Cloudflare Workers: Wrangler Configuration（`assets`フィールド）](https://developers.cloudflare.com/workers/wrangler/configuration/)
- [Migrate from Pages to Workers](https://developers.cloudflare.com/workers/static-assets/migration-guides/migrate-from-pages/)
- [Workers CI/CD: Builds](https://developers.cloudflare.com/workers/ci-cd/builds/)
- 根拠コミット: `1bdd484`（Pages想定の初期設定）, `64d37e9`（Workers向け修正）
- 根拠PR: [#2](https://github.com/kishimin/pi-lab/pull/2), [#3](https://github.com/kishimin/pi-lab/pull/3)
- 確認日: 2026-08-09
