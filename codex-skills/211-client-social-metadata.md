# ページ遷移時のソーシャルカード情報を更新する

[PR #223](https://github.com/kishimin-ai-create/profile-hub/pull/223) では、公開画面のページ別メタデータに Twitter Card の種類、タイトル、説明、任意の画像を追加した。画像があるページからないページへ移動すると、カード種別を `summary` に戻し、以前の画像タグを残さない。

公開画面の57テスト、整形、型チェック、Lint、HTML 検査、ビルドが成功した。タグは JavaScript の実行後に設定されるため、JavaScript を実行しないクローラー向けの初期 HTML は解決していない。prerender は別途必要で、PR はドラフトのままである。
