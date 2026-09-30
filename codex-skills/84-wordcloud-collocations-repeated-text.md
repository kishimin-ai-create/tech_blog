# WordCloudで同じ文字列が並ぶ理由をcollocationsから調べる

## 結論

WordCloud 1.9.6の`generate()`は、既定で`collocations=True`を使う。連続する同一トークンが統計的に強い2語の組と判定されると、その組を1つの描画項目として採用する。描画項目の文字列には両方のトークンが残るため、画像では同じ文字列が2つ並んで見える。

単語ごとに描画したい場合は`collocations=False`を指定する。これは語の表記を変えず、2語の組を作る処理を止める設定である。

## 発生した問題

形態素解析済みのトークンを空白で連結し、`WordCloud.generate()`へ渡す処理で、同じ文字列が画像内に並んで描画された。入力の末尾には同じトークンの連続があった。入力での反復は頻度を表すが、それだけでは画像中の2つの文字列を説明できない。

対象環境はPythonの`wordcloud==1.9.6`である。アプリ側は`services/analysis/analyze.py`で`wordcloud_terms`を連結し、`generate(wordcloud_text)`へ渡していた。

## 調査

保存済みの解析テキストをWordCloud 1.9.6の`process_text()`へ渡し、採用されたキーを確認した。`collocations=True`では、連続した同一トークンを含む2語のキーが現れた。`collocations=False`ではそのキーは現れず、単語のキーだけが残った。描画用の`layout_`でも、後者は該当する単語を1項目として配置した。

ライブラリの`process_text()`は`collocations=True`のとき`unigrams_and_bigrams()`を呼ぶ。この関数は隣接する2語を数え、スコアが`collocation_threshold`を超えた組を語句として登録する。既定のしきい値は30であり、語句の意味や日本語の人名かどうかは判定しない。

## 原因と対応

連続した同一トークンが2語の組として採用され、その2語を含む1つの描画項目になったことが原因だった。`repeat=False`でも語句自体の中に同じ文字列が2つあるため、画像上は重複して見える。

単語単位の集計を続ける場合の設定は次のとおり。

```python
word_cloud = WordCloud(collocations=False)
word_cloud.generate(wordcloud_text)
```

調査時には、実データを使ったメモリ上の比較で2語のキーが消えることを確認した。アプリの修正をコミットし、生成画像を再確認する工程はこの調査の対象外である。

## 学び

画像に同じ文字列が2つ見えたときは、入力中の出現回数だけで判断せず、`process_text()`が返す描画キーを確認する。形態素解析を先に済ませている処理では、WordCloud側の2語抽出が意図に合うかを明示的に決める。

## 参考資料

- [WordCloud 1.9.6の`process_text()`と`generate()`](https://github.com/amueller/word_cloud/blob/1.9.6/wordcloud/wordcloud.py)
- [WordCloud 1.9.6の2語判定](https://github.com/amueller/word_cloud/blob/1.9.6/wordcloud/tokenization.py)
