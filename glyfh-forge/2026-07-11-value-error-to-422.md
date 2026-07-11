# ValueErrorを422に変換する

画像生成では、空文字のような不正な入力を受けることがある。

内部では `ValueError` を投げる。

APIでは、それをHTTP 422として返す。

## なぜ422か

リクエストの形は届いている。

でも、中身の値が処理できない。

だから 500 ではなく 422 にする。

## 実装の場所

`app/main.py` の `_render_or_422` が、`ValueError` を `HTTPException(status_code=422)` に変換している。

