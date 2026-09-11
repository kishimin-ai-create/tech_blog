# 本番E2EでCORS再デプロイと5ブラウザーの画像生成を確認する

## 対象読者

Cloudflare Pagesのフロントエンドと、別ホストのAPIを組み合わせたWebアプリで、ローカルだけでなく本番相当の画像生成フローをPlaywrightで確認したい開発者を対象にする。

## スコープ

Mojicaの本番E2Eで行ったCORS再デプロイ後の確認、Glyph Forgeの待ち時間、英語ロケーターの不一致修正、5 worker・5ブラウザーでの最終結果を扱う。CloudflareやAppRunのデプロイ手順そのものは扱わない。

## 結論

本番E2Eは、画面が表示されることだけでなく、フロントエンドから別ホストのMojica APIへ実際にリクエストでき、APIがGlyph Forgeを経由してPNGを返し、ブラウザーがダウンロードできることまで確認する必要がある。MojicaではCORS再デプロイ後に本番URLから画像生成を確認し、並列実行時のGlyph Forge待ちを観測した。英語画面の実際のラベルに合わせてロケーターを修正した結果、5 worker・5ブラウザーで30件が成功した。

## 本番構成と確認範囲

Mojicaのリリース構成では、フロントエンドをCloudflare Pages、Mojica APIとGlyph Forge APIを別サービスとして配置する。ブラウザーはMojica APIだけを呼び、Mojica APIがGlyph Forge APIへ接続する。この構成では、ローカルの同一オリジンテストだけではCORSやデプロイ済み環境の接続不備を検出できない。

本番E2Eでは、次の入力を日本語・英語で実行した。

- 標準画像
- X背景画像
- Xアイコン画像

各ケースで入力、画像タイプ選択、送信、PNGファイル名、画面キャプチャを確認する。6ケース（3画像タイプ×2言語）を5ブラウザープロジェクトで実行するため、最終結果は30件になる。

## CORS再デプロイ後の確認

APIの許可オリジンを変更した場合、ソースコードや設定ファイルを修正しただけでは本番環境に反映されない。CORS設定を再デプロイした後、Cloudflare Pages上のフロントエンドから本番APIへ送信し、ブラウザーがレスポンスを受け取れることをE2Eで確認する。

この確認で重要なのは、APIへ直接リクエストして200になることではない。実際のページを開き、ブラウザーのOrigin付きリクエストとして画像生成を行い、CORSによってレスポンスがブロックされていないことを確認することである。

## 並列実行時のGlyph Forge待ち

5 workerで本番E2Eを実行した際、Glyph Forgeの処理待ちが発生した。これはテストコードの単純な操作待ちと同じではなく、画像生成という外部サービスの処理時間に依存する待ちである。

そのため、次の観点を分けて調査した。

| 観点 | 確認内容 |
| --- | --- |
| ブラウザー操作 | 入力・選択・送信が正しいか |
| Mojica API | CORSを通過してリクエスト・レスポンスできるか |
| Glyph Forge | 並列要求を処理できるか、待ちやレート制限がないか |
| ダウンロード | PNGレスポンスがブラウザーの保存処理まで届くか |

待ちが発生した事実だけでテストを失敗扱いにせず、サービス処理時間とテストタイムアウトの関係を確認する必要がある。

## 英語ロケーターの修正

英語版の実画面では、テキスト入力のラベルが`Text to render`だった。一方、E2Eセレクターは`Text to draw`を期待していたため、英語リリーステストの入力要素を取得できなかった。

修正後は、テストが想定した文言を追加するのではなく、デプロイ済み画面が実際に公開しているラベルへセレクターを合わせた。ロケーターは次のようにロケールごとの正規表現として管理している。

```ts
textLabel: {
  ja: /描画する文字列/,
  en: /Text to render/,
},
```

画面の仕様変更でラベルが変わる場合、実画面、i18n定義、E2Eセレクターを同じ変更として確認する必要がある。

## 最終結果

修正後、5 worker・5ブラウザープロジェクトで30件の本番E2Eが成功した。確認対象は、Cloudflare PagesからのAPI接続、3種類の画像生成、PNGダウンロード、日本語・英語の画面である。

## 事実・判断・未確認事項

- FACT: 本番E2E用に、デプロイ済みAPIへ接続するLargeテストを追加した。
- FACT: 画像タイプ3種と日本語・英語を組み合わせた6ケースを定義している。
- FACT: Playwright設定は5 workerと5ブラウザープロジェクトを使用する。
- FACT: CORS再デプロイ後、本番フロントエンドから画像生成を確認した。
- FACT: 5 workerの同時実行ではGlyph Forgeの処理待ちが発生した。
- FACT: 英語ロケーターの`Text to draw`を、実画面の`Text to render`へ修正した。
- FACT: 最終的に5 worker・5ブラウザー・30件が成功した。
- INFERENCE: 本番E2Eは、ブラウザー、API、外部画像生成サービス、ダウンロードの境界を分けて観測できるようにする必要がある。
- ASSUMPTION: Glyph Forgeの待ち時間が将来の負荷状況でも同じ範囲に収まるかは、継続的な実行結果で再評価する必要がある。

## まとめ

本番E2Eでは、ローカルテストで見えないCORS、デプロイ反映、外部サービス待ち、実際のロケーター不一致が現れる。これらをブラウザー操作・API接続・外部サービス・ダウンロードに分けて確認し、最終的に5 worker・5ブラウザーで30件を成功させた。

## 参考

- `mojica/docs/v1/release.md`
- `mojica/frontend/e2e/tests/image-generation.large.test.ts`
- `mojica/frontend/e2e/pages/image-generation-page.ts`
- `mojica/frontend/e2e/selectors/image-generation-selectors.ts`
- `mojica/frontend/playwright.config.ts`
- Mojica commit `02e1403 test: verify deployed image generation`
- Mojica commit `e62981a fix: align English release selector`
- Mojica commit `28633bf test: add diagnostic steps to e2e flows`

確認日: 2026-09-06
