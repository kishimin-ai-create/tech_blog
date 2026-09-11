# Cloudflare PagesでBunプロジェクトの自動npm installが失敗したときの切り分け

## 結論

Cloudflare Pagesの自動依存関係インストールが`npm install`を選び、Bunのロックファイルを使うViteプロジェクトで失敗した。`SKIP_DEPENDENCY_INSTALL=1`を設定して自動インストールを止め、ビルドコマンドで`bun install --frozen-lockfile`を実行すれば、プロジェクトが選んだパッケージマネージャーを明示できる。

## 発生した問題

MojicaのFrontendをCloudflare Pagesへデプロイした際、リポジトリのcloneは成功したが、ビルド前の依存関係インストールで停止した。

```text
Detected the following tools from environment: npm@10.9.2, nodejs@22.16.0
Installing project dependencies: npm install --progress=false
npm error Cannot read properties of null (reading 'edgesOut')
```

`No Wrangler configuration file found`も表示されたが、これは処理を継続する通知であり、終了原因ではなかった。

## 原因

プロジェクトは`frontend/bun.lock`を管理している一方、Pagesの自動処理は`npm install`を実行した。依存関係の導入方法がリポジトリのロックファイルと一致していなかったことが、今回の失敗条件だった。

## 解決方法

Pagesの設定を次のように変更した。

```text
Root directory: frontend
Build command: bun install --frozen-lockfile && bun run build
Build output directory: dist
SKIP_DEPENDENCY_INSTALL: 1
BUN_VERSION: 1.2.15
```

`SKIP_DEPENDENCY_INSTALL`でPagesの自動インストールを無効化し、Build commandでBunを明示的に呼び出す。`VITE_API_URL`にはデプロイ済みAPIの公開URLを設定する。

## 動作確認

設定変更後、Cloudflare Pagesで再デプロイし、ビルドが完了して公開サイトが生成された。CloudflareのBuild imageにはBunが用意され、依存関係の自動インストールを無効化する`SKIP_DEPENDENCY_INSTALL`も公式に提供されている。

## 未確認事項

今回の記録では、Pagesのビルドログ全文や本番APIへの画面操作までは保存していない。公開後のAPI接続は、Pagesの公開URLをMojica APIのCORS許可リストへ追加した状態で別途確認する必要がある。

## 参考資料

- [Cloudflare Pages Build image](https://developers.cloudflare.com/pages/configuration/build-image/)
- [Cloudflare Pages Build configuration](https://developers.cloudflare.com/pages/configuration/build-configuration/)
