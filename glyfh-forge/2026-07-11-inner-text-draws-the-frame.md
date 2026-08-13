# frameの内側はinner_textで描く

## はじめに

プロフィール画像のルールはシンプルだ。

`frame_text` の形の内側には `inner_text` を敷き詰める。

`frame_text` の文字そのものは出さない。

## 実装の見方

`render_x_icon_image` や `render_background_image` は、まずframe用のマスクを作る。

そのマスクの内側だけに `inner_text` の画像を残す。

これで「frame_textの形を、inner_textで描いた」状態になる。

## 小さな結論

frameは文字ではなく領域。

innerはその領域を埋める文字。

## まとめ

innerはその領域を埋める文字。
