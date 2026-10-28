# APIエラーとログに入力値を不用意に含めない

APIで例外が起きたとき、詳細をそのまま返すと内部情報が利用者へ漏れる可能性がある。一方で、失敗の種類が分からなければ呼び出し側は扱いにくい。

profile-hubでは、予期しない例外に汎用の500応答を返し、4xxには共通のクライアントエラーコードを用意した。リクエストログはメソッド、ステータス、処理時間に限定し、URLやヘッダーを記録しない。[PR #55](https://github.com/kishimin-ai-create/profile-hub/pull/55)、[#57](https://github.com/kishimin-ai-create/profile-hub/pull/57)、[#75](https://github.com/kishimin-ai-create/profile-hub/pull/75) に対応する変更がある。

[PR #76](https://github.com/kishimin-ai-create/profile-hub/pull/76) の検証ヘルパーは、許可したフィールド名を最大20件まで扱い、汎用のエラーメッセージを返す。入力値をエラーへそのまま含めないための境界である。ただし、当時このヘルパーを使う本番書き込みルートはなく、リクエストボディのサイズ制限も未実装だった。

安全な表示とログは、入力制限や認証の代わりにはならない。新しいエンドポイントを追加する際には、それぞれの境界を改めて検証する必要がある。
