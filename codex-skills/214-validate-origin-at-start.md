# 公開オリジンの誤りを起動時に検出する

## 課題

profile-hub の `SITE_ORIGIN` はサイトマップと canonical URL の生成に使う。開発サーバーでは不正な値が設定されても起動でき、ページのメタデータを描画する段階で初めてエラーになる場合があった。

## 対応

[#231](https://github.com/kishimin-ai-create/profile-hub/pull/231) では Vite 設定の読み込み時にオリジンを検証・正規化し、同じ値をクライアントと生成物へ渡す。未設定はローカル開発用として許容し、パスや認証情報を含む値は拒否する。

## 検証と限界

パスを含む `SITE_ORIGIN` で開発サーバーを起動すると、設定エラーを表示して終了した。正常なテスト用オリジンでビルドし、sitemap と robots.txt の検査が成功した。整形・型・Lint・HTML 検査と65テスト、[統合 CI](https://github.com/kishimin-ai-create/profile-hub/actions/runs/41617069641) の全5ジョブも成功。PR はドラフトであり、本番環境の設定や配信は未確認である。
