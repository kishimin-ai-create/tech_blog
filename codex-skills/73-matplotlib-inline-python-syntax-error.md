# `%matplotlib inline`で`Expected expression`になる原因と実行環境の分け方

## 結論

`%matplotlib inline`はPythonの文ではなく、Jupyter/IPythonが解釈するマジックコマンドである。通常の`.py`ファイルをPythonとして実行すると、`%`から始まる行をPython構文として解析するため、`Expected expression`の構文エラーになる。

通常のスクリプトではこの行を削除し、描画結果を表示する場合は`plt.show()`を使う。Notebookで実行する場合だけ、Notebookのセルで`%matplotlib inline`を使う。

## 発生した問題

### 症状

通常のPythonコードへ次の行を追加すると、エディターまたはPython実行時に次のエラーが出る。

```python
%matplotlib inline
```

```text
Expected expression
```

### 発生条件

- OS: Windows
- 実行対象: Pythonの`.py`スクリプト
- 使用した記法: Jupyter/IPythonマジックコマンド
- 目的: MatplotlibのグラフをNotebook内へ表示する

### 影響

Pythonファイル全体が構文として解釈できず、MeCabやWordCloudの処理まで実行されない。

## 調査

### 仮説1: Matplotlibのインストールに失敗している

エラーに`matplotlib`が含まれるため、最初にライブラリのインストール問題を疑いやすい。しかし、このエラーは`import matplotlib`より前に発生する構文解析エラーである。

```python
import matplotlib.pyplot as plt
```

ライブラリの有無を確認する`ModuleNotFoundError`とは種類が異なる。

### 仮説2: 実行場所がJupyterではない

`%matplotlib inline`の`%`は、通常のPython演算子として使われているのではない。Jupyter/IPythonがセルを実行する前に解釈する特別な記法である。

通常のPythonへ渡すと、Pythonの文法として有効な式にならないため、`Expected expression`になる。

なお、調査時点の`blog-word-cloud/word-cloud.py`には`%matplotlib inline`の行は存在しなかった。このエラーが出た場合は、Notebookのセル、エディターの実行対象、または未保存の編集内容が、実際に実行したファイルと一致しているかも確認する必要がある。

## 原因

`%matplotlib inline`を、Jupyter/IPythonセルではなく通常のPythonコードとして解釈させたことが原因である。

`%matplotlib inline`はPython標準構文ではない。したがって、`.py`ファイルを`python word-cloud.py`で実行する場合には書けない。

## 解決方法

### `.py`ファイルとして実行する場合

マジックコマンドを削除し、Matplotlibの描画を表示する箇所で`plt.show()`を呼び出す。

```python
import matplotlib.pyplot as plt

plt.imshow(wc)
plt.axis("off")
plt.show()
```

画像ファイルへ保存するだけなら、`plt.show()`も不要である。

```python
wc.to_file("sample.png")
```

### Jupyter Notebookとして実行する場合

Notebookのセルであれば、次の行を使える。

```python
%matplotlib inline
```

この場合は、ファイルを通常のPythonスクリプトとして実行するのではなく、Jupyterカーネルへセルを送る。

## 動作確認

通常のPythonコンパイラーへマジックコマンドを渡すと、構文エラーになることを確認できる。

```powershell
python -c "compile('%matplotlib inline', '<string>', 'exec')"
```

この確認で、問題がMatplotlibのimportや描画バックエンドより前の、Python構文解析段階にあることが分かる。

調査時点の`word-cloud.py`は`matplotlib.pyplot`をimportし、WordCloudを画像へ保存する構成だった。`%matplotlib inline`がファイルにないことも確認したため、実際にエラーが発生した実行対象が同じファイルか、Notebookセルかを追加で確認する必要がある。

## 再発防止

- `.py`ではJupyterマジックを使わない。
- Notebookで使うコードと、通常のPythonスクリプトで使うコードを分ける。
- エラーがエディター由来かPython実行時かを分けて確認する。
- 実行前に、実行対象ファイルと保存済みの内容が一致していることを確認する。

## まとめ

- `%matplotlib inline`はJupyter/IPython専用のマジックコマンドである。
- 通常の`.py`へ書くと、Matplotlibのインストール前に構文エラーになる。
- `.py`では`plt.show()`または`WordCloud.to_file()`を使う。
- Notebookで実行する場合だけ、セル内で`%matplotlib inline`を使う。

