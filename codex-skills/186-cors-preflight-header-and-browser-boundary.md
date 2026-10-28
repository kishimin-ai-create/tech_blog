# CORSのプリフライトと通常のOPTIONSを区別する

異なるオリジンの管理画面からAPIを呼ぶとき、ブラウザは条件によって本リクエストの前にプリフライトを送る。通常のOPTIONSまで同じものと扱うと、ルーティングやエラーの挙動を混同する。

[PR #73](https://github.com/kishimin-ai-create/profile-hub/pull/73) では `Access-Control-Request-Method` を持つOPTIONSをプリフライトとして扱い、許可された `PUBLIC_ORIGIN` と `ADMIN_ORIGIN` に応答する。許可していないオリジンは拒否し、Originのないリクエストはブラウザ外の利用も考慮する。レスポンスにはキャッシュがオリジンごとに異なることを示す `Vary` を付ける。

Chromiumを使い、許可オリジンからのAuthorizationヘッダーを含むプリフライトと、許可されないオリジンの遮断を確認した。これはChromiumで確認した範囲の結果であり、すべてのブラウザでの検証ではない。

CORSはブラウザによる読み取り制御であって、認証や認可ではない。この変更ではCookieを伴うCORSも実装しておらず、将来Cookieを使う場合は別途設計が必要になる。
