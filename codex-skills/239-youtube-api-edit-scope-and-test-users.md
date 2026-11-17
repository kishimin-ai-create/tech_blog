# YouTube Data APIで動画の概要欄を編集するときの認証権限

## はじめに

操作動画のアップロード後、概要欄のリンクを公開サイトのURLへ変更しようとした。アップロード時に使った認証では編集が拒否され、必要なスコープで再認証した。

## 前提・環境

- YouTube Data API v3と、デスクトップアプリ向けOAuthクライアントを使用
- OAuthアプリは外部ユーザー向けのテスト状態
- 認証情報やトークンはリポジトリに保存しない

## やってみた結果

アップロードと読み取りのスコープで取得した認証を使い、`videos.update`で概要欄を変更しようとすると、HTTP 403と`ACCESS_TOKEN_SCOPE_INSUFFICIENT`が返った。`youtube.force-ssl`で再認証すると更新はHTTP 200で成功した。更新後に動画情報を再取得し、概要欄の公開URLを確認した。

## 実装・検証

### テストユーザーを登録して認証する

最初の認証では、テスト中のアプリへ許可されていないユーザーとして403 `access_denied`が表示された。Google CloudのOAuth設定で対象アカウントをテストユーザーに追加し、認証をやり直した。

### 操作に必要なスコープを確認する

動画の投稿には`youtube.upload`と`youtube.readonly`を指定していた。後から概要欄を変更する際は、それらを使った認証で権限不足となった。新たに動画の編集・削除を許可する`youtube.force-ssl`を要求し、利用者の同意を得た。

更新時は動画の既存`snippet`からタイトル、カテゴリ、言語などを引き継ぎ、説明だけを変更した。更新後はAPIで内容を再取得した。旧動画の削除は利用者の明示的な依頼を受けて実行し、HTTP 204の後に再取得できないことを確認した。

## つまずいたところ

アップロードが成功したことは、後続の編集操作に必要な権限がある証拠にはならなかった。既存の同意範囲と、個別のAPI操作が受け付けるスコープを照合する必要があった。

## 学んだこと

- テスト中の外部OAuthアプリでは、認証に使うアカウントのテストユーザー登録を確認する。
- 操作を追加するときは、必要なスコープと付与済みの権限を確認する。
- 動画の削除は不可逆な操作として対象IDを確認し、結果も再取得する。

## まとめ

今回の更新は再認証後に完了した。必要な権限は操作単位で調べ、利用者に認可を求める範囲を理解してもらうことが重要である。

## 参考資料

- [YouTube Data API: Authorization](https://developers.google.com/youtube/data/guides/authentication)
- [YouTube Data API: Videos update](https://developers.google.com/youtube/v3/docs/videos/update)
- [Google OAuth: App state overview](https://developers.google.com/identity/protocols/oauth2/production-readiness/overview)
