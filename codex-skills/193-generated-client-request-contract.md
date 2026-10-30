# 生成 API クライアントのヘッダーを通信境界で確認する

OpenAPI からクライアントを生成しても、ブラウザーが必要なヘッダーと Cookie を送るかは、実際の通信経路で確認する必要がある。profile-hub の管理 API はブラウザー認証の変更要求に `X-Admin-Request: 1` を要求する。

## 対応

[PR #116](https://github.com/kishimin-ai-create/profile-hub/pull/116) では、Axios の共通通信処理が対象のブラウザー認証 POST にヘッダーを加え、`withCredentials` を有効にした。対象は login、refresh、logout の3経路に限定し、通常の通信に資格情報を無条件で付けない。

[PR #117](https://github.com/kishimin-ai-create/profile-hub/pull/117) では生成した login 関数を MSW の受信側まで通し、ヘッダーとレスポンスを確認した。生成された型の存在だけでは、送信される HTTP リクエストまでは確認できないためである。

## 検証と限界

管理画面の `src/api/generated-auth.small.test.ts` は、生成関数から送られた `X-Admin-Request` を検証する。共通通信処理のテストも対象経路の設定を確認する。

これらのテストは実ブラウザーでのサードパーティ Cookie の許可を証明しない。CORS、Cookie の送受信、ブラウザー固有の制限は実際の配信構成で確認する必要がある。変更は積み重ね PR 上にあり、本番反映を意味しない。
