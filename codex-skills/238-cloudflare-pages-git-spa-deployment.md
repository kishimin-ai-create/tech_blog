# Cloudflare PagesのGit連携でSPAの直接アクセスを確認する

## はじめに

Workersの静的配信で公開していたVite製SPAを、GitHub連携のCloudflare Pagesでも公開した。Pagesプロジェクトの作成だけで公開済みと判断せず、本番ビルドと深いURLからの操作を確認した。

## 前提・環境

- Vite・React製の静的SPA。GitHubの`main`を本番ブランチとする
- ビルドコマンドは`npm run build`、出力先は`dist`
- 既存のWorkers配信と`wrangler.jsonc`は残した

## やってみた結果

GitHub連携のPagesプロジェクトを作成し、コミット`225943c`を本番デプロイした。Cloudflare APIのデプロイ段階は`success`、`/`と`/pi-message`のHTTPステータスは200だった。Playwrightで直接`/pi-message`を開き、入力、結果表示、再試行を確認した。

## 実装・検証

### 公開方式とビルド設定

既存のPagesプロジェクトとGit連携の設定を確認したうえで、新しいプロジェクトを`main`に接続した。Direct Uploadで作ったプロジェクトは後からGit連携へ切り替えられないため、公開方式は作成前に決める必要がある。

ローカルでは`npm run build`が成功した。Pages側も同じコマンドでビルドし、`dist`を公開した。プロジェクトの作成と初回ビルドにはCloudflare APIを使用した。Git連携は設定されたが、その後の新しいpushによる自動デプロイは試していない。

### 深いURLと画面操作を確認する

本番デプロイの進行中、`/`と`/pi-message`は一時的にHTTP 522を返した。その後デプロイが成功し、両方のパスでHTTP 200を確認した。HTTP応答だけではReactの動作まで分からないため、Playwrightでメッセージを入力して結果を表示し、再試行後の入力欄まで確認した。

## つまずいたところ

デプロイ直後の522は、デプロイがまだ進行中だった時点で観測された。原因を特定したわけではないため、両者の因果関係は断定しない。

## 学んだこと

- PagesのGit連携とDirect Uploadはプロジェクト作成前に選ぶ。
- Cloudflareのデプロイ結果、HTTP応答、ブラウザー操作を別々に確認する。
- Pagesでは404ページを置かないSPAに対し、静的アセットに一致しないパスをルートへフォールバックできる。

## まとめ

Pages本番で深いURLから操作できた。既存のWorkers配信は削除しておらず、切り替え後の案内や旧URLの扱いは別途判断が必要である。

## 参考資料

- [Cloudflare Pages: Git integration](https://developers.cloudflare.com/pages/configuration/git-integration/)
- [Cloudflare Pages: Direct Upload](https://developers.cloudflare.com/pages/get-started/direct-upload/)
- [Cloudflare Pages: Serving Pages](https://developers.cloudflare.com/pages/configuration/serving-pages/)
