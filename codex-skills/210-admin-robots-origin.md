# 管理画面の配信元に robots.txt を置く

管理画面が公開サイトと別の配信元に置かれる構成では、管理画面のクロール方針は管理画面の配信元で返す必要がある。[PR #222](https://github.com/kishimin-ai-create/profile-hub/pull/222) では管理アプリの `public/robots.txt` に全クローラー向けの `Disallow: /` を追加した。

管理画面のビルドと整形チェックが成功し、生成された `dist/robots.txt` の内容を確認した。実際の配備先でこのファイルが返ることは未確認である。robots.txt はクローラーへの指示であり、認証やアクセス制御の代替ではない。PR はドラフトで未マージ。
