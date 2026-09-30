# 形態素解析済みの語をWordCloudへ渡す2つの方法

## 結論

WordCloud 1.9.6の`generate(text)`は、渡された文字列をWordCloud自身が再分割して頻度を作る。`generate_from_frequencies(frequencies)`は、呼び出し側が作った単語と頻度の対応を直接受け取る。

形態素解析で語の境界を既に決め、その境界と出現回数を描画に使いたいなら、`Counter`で集計して`generate_from_frequencies()`へ渡す方法が合う。ただし、この経路ではWordCloud側のstopwords処理とcollocations処理を通らないため、必要な除外は呼び出し側で行う。

## 比較する目的と前提

対象のPython処理は、SudachiPyで抽出したトークンを`wordcloud_terms`へ集めてからWordCloudへ渡す。元の経路はトークンを空白で連結し、`generate()`へ渡していた。比較の目的は、形態素解析済みの語をWordCloudが再解釈するかどうかを明確にすることにある。

対象ライブラリは`wordcloud==1.9.6`。ここでは画像の見栄えや実行速度は比較していない。

## 比較

| 評価軸 | `generate(text)` | `generate_from_frequencies(frequencies)` |
| --- | --- | --- |
| 入力 | 文字列 | 単語をキー、出現回数を値とする対応 |
| 語の分割 | `process_text()`で再分割する | 入力キーをそのまま使う |
| 連続語の判定 | `collocations=True`なら実施する | 実施しない |
| WordCloudのstopwords | `process_text()`で適用する | 適用しない |
| 用途 | 生の文章を渡す | 語と頻度が既に決まっている |

WordCloud 1.9.6の実装では、`generate()`は`generate_from_text()`を経由し、`process_text()`の結果を`generate_from_frequencies()`へ渡す。後者を直接呼ぶと、その前処理経路を省く。

## 形態素解析済みの語を直接渡す場合

```python
from collections import Counter

filtered_terms = [term for term in wordcloud_terms if term not in stop_words]
word_cloud.generate_from_frequencies(Counter(filtered_terms))
```

この例はAPIの使い方を示すもので、アプリに適用済みの差分ではない。既存のstopwords設定を維持したいなら、頻度を作る前に同等の除外規則を実装する必要がある。大文字小文字や表記の統合も必要なら呼び出し側の責任になるが、今回の調査では表記の正規化を提案しない。

## 検証と制約

WordCloud 1.9.6のインストール済み実装で、`generate()`が`process_text()`を通ること、`generate_from_frequencies()`が受け取ったキーを直接描画候補にすることを確認した。実データによる`collocations=False`の比較も実施した。一方、この頻度入力への置換後にアプリ全体を実行した結果は未確認である。

## 参考資料

- [WordCloud 1.9.6の生成APIと引数](https://github.com/amueller/word_cloud/blob/1.9.6/wordcloud/wordcloud.py)
