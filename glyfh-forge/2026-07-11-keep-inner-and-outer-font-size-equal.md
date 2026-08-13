# innerとouterの文字サイズを揃える

## はじめに

`inner_text` と `outer_text` の文字サイズが違うと、画像の密度が変に見える。

特にプロフィール画像では、外側だけ大きい、内側だけ小さい、という差がすぐ目立つ。

## 今回の方針

プロフィール画像では、innerもouterも同じ `output_font_size` で描く。

`render_text_grid_image` に渡すfont sizeを揃えるだけでも、見た目の安定感がかなり変わる。

## テストで見ること

テストでは、inner用とouter用の描画呼び出しが同じfont sizeになることを確認している。

## まとめ

テストでは、inner用とouter用の描画呼び出しが同じfont sizeになることを確認している。
