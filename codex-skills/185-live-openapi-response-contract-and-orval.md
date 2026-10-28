# 実際の公開APIに合わせたOpenAPIをOrvalで確認する

OpenAPIからクライアントを生成できても、仕様が実際のレスポンスと一致するとは限らない。profile-hubでは、お気に入り、ポートフォリオ、プロフィールの公開APIを追加した後、各レスポンスとエラーを `/docs/json` に反映した。

プロフィールではリンクをグループ化した配列、ポートフォリオではnullになり得るURL、エラーではコードのリテラル値を表現した。生成された仕様からOrval 8.28.1でクライアントを作り、TypeScriptの型チェックを行った。[PR #70](https://github.com/kishimin-ai-create/profile-hub/pull/70)、[#71](https://github.com/kishimin-ai-create/profile-hub/pull/71)、[#72](https://github.com/kishimin-ai-create/profile-hub/pull/72)、[#78](https://github.com/kishimin-ai-create/profile-hub/pull/78) がそれぞれの契約を示す。

以前の記事では[固定フィクスチャからの生成と再試行](./176-orval-generation-compile-and-retry-check.md)を確認した。今回は実API側が公開する仕様を対象にした点が異なる。

型生成と型チェックは仕様を利用できることの確認であり、全入力に対するサーバー実装との一致やデプロイ先での動作を保証するものではない。手書きのスキーマは実装変更時に一緒に更新し、実レスポンスとのテストを保つ必要がある。
