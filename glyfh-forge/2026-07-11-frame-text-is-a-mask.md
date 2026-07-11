# frame_textは文字として描かない

Glyph Forgeのプロフィール画像では、`frame_text` をそのまま黒文字で描かない。

`frame_text` は、どこに `inner_text` を残すかを決めるためのマスクとして使う。

その結果、完成画像に見えるのは `inner_text` と `outer_text` だけになる。

## なぜそうするか

`frame_text` 自体を描くと、黒や別の色が混ざってしまう。

でも欲しいのは「frame_textの形」だ。

だから、`frame_text` は形だけ借りる。

## 覚えておくこと

`frame_text` は表示する文字ではなく、表示領域を作るための材料。

