# `MeCab`が見つからない原因を`pip`と`python`の不一致から切り分ける

## 結論

`mecab-python3`はインストール済みだったが、インストール先のAnacondaと、実行に使われた`slope-collector`の仮想環境が別だった。そのため、`pip`はパッケージを見つけられる一方、`python word-cloud.py`は`ModuleNotFoundError: No module named 'MeCab'`になった。

実行するPythonをインストール先と揃え、VS CodeのPython環境自動有効化を無効にする設定を追加した。

## 発生した問題

### 症状

`word-cloud.py`は起動時に`MeCab`をimportするだけの小さなスクリプトだった。

```python
import MeCab

mecab = MeCab.Tagger()
```

実行すると、次のエラーになった。

```text
ModuleNotFoundError: No module named 'MeCab'
```

### 発生条件

- OS: Windows
- シェル: PowerShell
- 実行対象: `blog-word-cloud/word-cloud.py`
- Python: `slope-collector/.venv/Scripts/python.exe`
- `pip`: Anacondaの`pip.exe`
- インストール済みパッケージ: `mecab-python3 1.0.12`

### 影響

MeCabの形態素解析を実行できなかった。ただし、MeCabの辞書や`Tagger`の初期化が壊れていたのではなく、実行Pythonからモジュールが見えない状態だった。

## 調査

### 仮説1: `mecab-python3`がインストールされていない

まず、`pip`の情報を確認した。

```powershell
pip show mecab-python3
```

結果は次のとおりだった。

```text
Name: mecab-python3
Version: 1.0.12
Location: C:\Users\<user>\anaconda3\Lib\site-packages
```

したがって、パッケージはインストール済みだった。この仮説は否定できる。

### 仮説2: `python`と`pip`が別の環境を指している

実体を確認した。

```powershell
Get-Command python,pip -All
python -c "import sys; print(sys.executable)"
python -m pip --version
```

確認できた対応関係は次のとおりだった。

| コマンド | 実体 |
| --- | --- |
| `python` | `slope-collector/.venv/Scripts/python.exe` |
| `pip` | `anaconda3/Scripts/pip.exe` |
| `python -m pip` | `No module named pip` |

`python`の`sys.path`にはAnacondaの`site-packages`が含まれないため、Anacondaへ入れた`MeCab`をimportできなかった。

### 仮説3: MeCab本体や辞書の初期化に失敗している

AnacondaのPythonを明示して、同じスクリプトを実行した。

```powershell
C:\Users\<user>\anaconda3\python.exe -X utf8 .\word-cloud.py
```

形態素解析結果が出力され、`MeCab.Tagger()`の初期化も成功した。この仮説も今回の原因ではなかった。

## 原因

原因は、`pip install mecab-python3`を実行したPythonと、スクリプトを実行したPythonが一致していなかったことだった。

さらに、現在のPowerShellプロセスには次の環境変数が残っていた。

```text
VIRTUAL_ENV=C:\Users\<user>\Desktop\programming\Python\slope-collector\.venv
```

この仮想環境は`slope-collector`の`uv sync`で作られる正規の環境であり、存在そのものは異常ではない。しかし、それを有効にしたターミナルを`blog-word-cloud`でも使い続けたため、別プロジェクトへ実行環境が持ち込まれた。

## 解決方法

### 実行環境を揃える

パッケージを入れたPythonと同じPythonで実行する。

```powershell
python -m pip install mecab-python3
python -X utf8 .\word-cloud.py
```

この`python`が意図したAnacondaを指していることを、作業前に確認する。

```powershell
python -c "import sys; print(sys.executable)"
```

### VS Codeの自動仮想環境有効化を止める

別プロジェクトの仮想環境を新しいターミナルへ自動注入しないため、VS Codeユーザー設定に次を追加した。

```json
{
  "python.defaultInterpreterPath": "C:\\Users\\<user>\\anaconda3\\python.exe",
  "python.terminal.activateEnvironment": false
}
```

この設定は新しく作るターミナルに適用される。設定変更前から存在するターミナルは環境変数を保持するため、ターミナルの終了・再作成、またはVS Codeの再起動が必要になる。

## 動作確認

設定変更前の実行では、次のエラーを再現できた。

```text
python word-cloud.py
ModuleNotFoundError: No module named 'MeCab'
```

AnacondaのPythonを使った実行では、MeCabの解析結果が出力された。

```powershell
C:\Users\<user>\anaconda3\python.exe -X utf8 .\word-cloud.py
```

また、VS Code設定には次の値が保存されたことを確認した。

```text
python.defaultInterpreterPath = C:\Users\<user>\anaconda3\python.exe
python.terminal.activateEnvironment = false
```

ただし、既存のターミナルを再作成した後に、同じ`python`コマンドだけで実行できることはこの調査セッション内では未確認である。

## 再発防止

- `pip`単体ではなく、必ず`python -m pip`を使い、インストール先と実行先を揃える。
- `Get-Command python,pip -All`と`python -c "import sys; print(sys.executable)"`で実体を確認する。
- 別プロジェクトの仮想環境を有効にしたターミナルを、他のプロジェクトで使い回さない。
- VS Codeでは`python.terminal.activateEnvironment`を無効にし、自動的な環境混入を防ぐ。

## まとめ

- `MeCab`が見つからない原因は、未インストールではなくPython環境の不一致だった。
- `pip`が成功しても、実行する`python`が別ならimportできない。
- `python -m pip`で実行Pythonとインストール先を結び付けると、切り分けやすい。
- 仮想環境自体が悪いのではなく、別プロジェクトの環境を持ち込まない境界が必要だった。

