# SEO の生成物を CI で確認する

## 課題

profile-hub の sitemap と robots.txt は `SITE_ORIGIN` を指定したビルドで生成される。通常の CI ビルドにはこの変数がなく、生成処理の単体テストが成功しても、実際のビルド成果物は検証されていなかった。

## 対応

[#228](https://github.com/kishimin-ai-create/profile-hub/pull/228) では公開画面の夜間 CI ビルドにテスト用オリジンを与え、生成された sitemap の5 URL と robots.txt の参照先を検査する。検査スクリプトは Node 用 TypeScript 設定の対象にも含めた。

## 検証と限界

未設定ビルドの成果物に対して検査が失敗することを確認し、テスト用オリジンを設定したビルドでは成功した。整形・型・Lint・HTML 検査と64テストも成功し、[統合 CI](https://github.com/kishimin-ai-create/profile-hub/actions/runs/41615930662) で生成物の検査ステップが実行された。これはテスト用オリジンでの結果であり、実際の公開環境に正しい URL が設定されている証拠ではない。
