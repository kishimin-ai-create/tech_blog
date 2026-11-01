# Secure Cookie のブラウザーテストを HTTPS で確かめる

## 結論

管理画面のセッション復元は、実ブラウザーと HTTPS を組み合わせて確認した。ローカルの HTTP 環境では WebKit に Cookie が保存されても、復元リクエストには付かず 401 になった。自己署名証明書を使う HTTPS 環境では、Chromium と WebKit の両方でログイン、再読み込み後の復元、ログアウトを確認できた。

## 発生した問題

API は `HttpOnly; Secure; SameSite=None` の更新用 Cookie を返す。HTTP の localhost で実行した WebKit のテストでは、Cookie の保存を確認できた一方、更新リクエストは 401 を返した。Cookie を保存したことだけでは、その後の通信に Cookie が送られることを確認できなかった。

## 調査と対応

Cookie の保存と送信を別々に観察し、テスト用 API と画面の両方を `https://localhost` の別ポートで起動した。証明書はテスト用に一時生成し、ブラウザー設定でその自己署名証明書を許可する。実際の認証ルートを通し、画面の再読み込み後にセッションを復元する。

本番設定で証明書検証を無効にするための変更ではない。テスト内の設定に限っている。

## 動作確認

`apps/admin-web` で `bun run e2e:auth` を実行し、Chromium と WebKit でそれぞれログインから復元、ログアウトまでのシナリオが通った。未保存入力の離脱警告も両ブラウザーで実行し、合計 4 件が成功した。

API は実際の Hono の認証ルートを使うが、アカウントとセッション状態はテスト用のメモリー実装である。PostgreSQL を含む経路の証明ではない。また、localhost の異なるポートは同一サイトなので、異なるサイト間の第三者 Cookie 制限や本番ドメインでの動作も未確認である。

## 学び

Cookie の属性確認、保存確認、次のリクエストへの送信確認は別の観測点である。ブラウザーの認証フローでは、モックによる応答確認に加え、実際の HTTPS 通信を使って復元まで試すと差異を見つけやすい。

根拠: [ブラウザー認証テストの PR](https://github.com/kishimin-ai-create/profile-hub/pull/143)、[離脱警告テストの PR](https://github.com/kishimin-ai-create/profile-hub/pull/144)。
