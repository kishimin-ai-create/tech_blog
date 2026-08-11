# `wrangler deploy`と`wrangler versions upload`は別物——Workers Buildsのブランチ別デプロイコマンド

## 結論

Cloudflare Workers Builds（Git連携によるWorkerのビルド・デプロイ）は、本番ブランチと非本番ブランチで異なるデプロイコマンドを既定値として使う。本番ブランチは`wrangler deploy`（アップロードしたバージョンを即座に本番トラフィックへ昇格させる）、非本番ブランチは`wrangler versions upload`（プレビュー用のバージョンを作るだけで、本番には一切影響しない）。この2つのコマンドの違いを理解していないと、「デプロイされたのに本番に反映されない」あるいはその逆の誤解が起きやすい。

## 対象読者

- Cloudflare Workers Buildsのダッシュボード設定画面で、`wrangler deploy`と`wrangler versions upload`が別々の欄に表示されているのを見て、両者の違いが気になった人
- Cloudflareの「Gradual Deployments（段階的デプロイ）」の仕組みを理解したい人

## 3つのコマンドの違い

Cloudflare公式ドキュメント（`developers.cloudflare.com/workers/wrangler/commands/workers/`）で確認した、それぞれのコマンドの役割は次の通り。

| コマンド | 説明 | 本番トラフィックへの反映 |
| --- | --- | --- |
| `wrangler deploy` | Workerをデプロイする | 即座に反映される |
| `wrangler versions upload` | 新しいバージョンをアップロードするが、即座にはデプロイしない | 反映されない（別途`versions deploy`が必要） |
| `wrangler versions deploy` | 事前にアップロードしたバージョンを、一括または段階的にデプロイする | このコマンドの実行時に反映される（100%一括、または割合を指定した段階的ロールアウトも可能） |

つまり、`wrangler versions upload`だけを実行した状態は、「コードはCloudflareに送られているが、まだ誰にも配信されていない」状態である。

## Workers Buildsでの既定値

`developers.cloudflare.com/workers/ci-cd/builds/configuration/`によれば、Workers Builds（Git連携ビルド）は、ブランチの種類によって既定のデプロイコマンドを自動的に切り替える。

- **本番ブランチ**: 既定で`npx wrangler deploy`。アップロードされたバージョンは自動的にActive Deploymentへ昇格する。
- **非本番ブランチ**（PRブランチなど）: 既定で`npx wrangler versions upload`に置き換わり、本番へ昇格しないプレビュー用バージョンを作成するだけになる。

`developers.cloudflare.com/workers/ci-cd/builds/`には次のように明記されている（意訳）。

> ビルドが`wrangler deploy`のようなデプロイを行う設定の場合、アップロードされたバージョンは自動的にActive Deploymentへ昇格する。ビルドを自動実行させつつ本番への自動反映だけを無効にしたい場合は、デプロイコマンドを`npx wrangler versions upload`に変更する。

pi-labのCloudflareダッシュボード設定を確認したところ、メインのセットアップ画面には本番ブランチ用として`npx wrangler deploy`、「詳細設定」内には非本番ブランチ用として`npx wrangler versions upload`が表示されており、これは上記の公式な既定値と一致していた。

## なぜこの区別が必要か

PRごとに作られるプレビュー環境が、誤って本番に影響を与えないようにするためである。非本番ブランチのビルドが`wrangler deploy`のままだと、PRのコードがマージ前に本番トラフィックへ昇格してしまう。逆に、本番ブランチが`wrangler versions upload`のままだと、`main`へマージしても本番サイトが更新されない。

## 未確認事項

ダッシュボード上でどちらのコマンドがどちらのブランチ設定欄に対応しているかは、画面のテキストが報告された順序から推測したものであり、実際のUIレイアウト（スクリーンショット等）で直接確認したわけではない。確実に確認するには、Cloudflareダッシュボードの該当プロジェクト設定画面で、各欄のラベル（「本番ブランチのデプロイコマンド」「プレビューデプロイコマンド」等）を直接確認する必要がある。

## 参考資料

- [Wrangler Commands: Workers](https://developers.cloudflare.com/workers/wrangler/commands/workers/)
- [Workers CI/CD: Builds](https://developers.cloudflare.com/workers/ci-cd/builds/)
- [Workers CI/CD: Builds Configuration](https://developers.cloudflare.com/workers/ci-cd/builds/configuration/)
- [Gradual Deployments](https://developers.cloudflare.com/workers/configuration/versions-and-deployments/gradual-deployments/)
- 確認日: 2026-08-09
