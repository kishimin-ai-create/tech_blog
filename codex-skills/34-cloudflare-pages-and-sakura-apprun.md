# ViteのFrontendをCloudflare Pages、APIをさくらのクラウドAppRunへ分ける判断

## 結論

Mojicaでは、静的なVite FrontendをCloudflare Pagesへ、コンテナで動くMojica APIとGlyph ForgeをさくらのクラウドAppRunへ配置した。静的配信とコンテナ実行の責務を分けることで、Frontend用のコンテナとレジストリを追加せずに済む。

## 背景

Mojica FrontendはViteでビルドされ、成果物は`dist`に生成される。APIと画像生成サービスはASP.NET CoreおよびFastAPIのコンテナとしてAppRunへデプロイした。

## 選択肢

| 評価軸 | Cloudflare Pages | AppRunでFrontendも実行 |
| --- | --- | --- |
| 配信形式 | 静的ファイル | コンテナ |
| 追加レジストリ | 不要 | 必要 |
| API配置との統一 | APIとは分離 | 同一サービス |
| 観測した料金要素 | 静的アセットは無料・無制限 | インスタンスのvCPU・メモリ時間課金 |
| 運用 | Git連携と自動ビルド | イメージpushとアプリ設定 |

## 最終判断

FrontendはCloudflare Pages、APIはAppRunとした。Pagesでは`VITE_API_URL`をビルド時に設定し、公開後のPages URLをAPIのCORS許可リストへ追加する。

## 実装後の結果

Cloudflare Pagesのビルド設定を修正後、Frontendのデプロイに成功した。AppRun側では、Mojica APIイメージとGlyph Forgeイメージをさくらのコンテナレジストリから取得する構成にした。

## トレードオフ・今後の懸念

FrontendとAPIが異なるサービスになるため、CORS設定とAPI公開URLの管理が必要になる。Frontendのビルド時にAPI URLを埋め込むため、API URLを変更した場合はFrontendの再ビルドが必要である。

## 参考資料

- [Cloudflare Pages料金](https://developers.cloudflare.com/pages/functions/pricing/)
- [Cloudflare Pages Build configuration](https://developers.cloudflare.com/pages/configuration/build-configuration/)
- [さくらのクラウド AppRun共用型](https://cloud.sakura.ad.jp/products/apprun-shared/)
