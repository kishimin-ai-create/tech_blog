# Qiita Previewの`localeCompare`例外を直す：テンプレートを`public`の外へ置く

## 結論

Qiita Previewで`Cannot read properties of undefined (reading 'localeCompare')`が表示される場合、`public`配下に記事用Front Matterを持たないMarkdownがないか確認する。この記事の事例では、未完成テンプレートがタイトルなしの記事として一覧に混入していたため、`templates/`へ移動した。

## 発生した問題

Preview画面のサイドバーを表示すると、次の例外でアプリケーションが停止した。

```text
Unexpected Application Error!
Cannot read properties of undefined (reading 'localeCompare')
```

スタックトレースには、サイドバーの記事一覧をソートする処理と`Array.sort`が含まれていた。

## 調査

### 仮説1: 公開・下書き記事のタイトルが未設定

Previewの`/api/items`を取得し、一覧を調べた。`public/.remote/error.md`、`public/.remote/qiita.md`、`public/.remote/review.md`に対応する3項目には`title`がなかった。

```powershell
$items = Invoke-RestMethod -Uri 'http://localhost:8888/api/items'
($items.private + $items.draft + $items.public) |
  Where-Object { [string]::IsNullOrWhiteSpace($_.title) }
```

### 仮説2: 隠しディレクトリなら記事収集から除外される

3ファイルは`.remote`という隠しディレクトリにあったが、`public`の配下だった。API一覧にはこれらが下書きとして含まれていたため、この環境では隠しディレクトリ名だけでは除外されないと判断した。

## 原因

Qiita CLIが`public`配下のMarkdownを記事候補として読み込んだ結果、記事用Front Matterを持たないテンプレートのタイトルが`undefined`になった。サイドバーのソート処理がその値へ`localeCompare`を呼び出し、ブラウザ側で例外になった。

## 解決方法

テンプレートの本文を変更せず、`public/.remote/`から`templates/`へ移動した。

```text
templates/
  error.md
  qiita.md
  review.md
```

`templates/`はQiita CLIの公開対象である`public`の外にあるため、一覧APIには現れない。

## 動作確認

移動後に一覧APIを再取得し、22件のすべての項目にタイトルがあることを確認した。PreviewルートもHTTP 200を返した。

```powershell
Invoke-WebRequest -Uri 'http://localhost:8888/' -UseBasicParsing
```

この確認後、調査用に起動していたPreviewプロセスは停止した。

## 再発防止

- Qiitaに投稿しないテンプレートやメモは`public`の外に置く。
- `public`へ追加したMarkdownには、Qiita CLIが要求するFront Matterを設定する。
- `Unexpected Application Error!`だけで判断せず、`/api/items`でタイトル欠落の項目を確認する。

## まとめ

- 症状: 一覧表示時に`localeCompare`で`undefined`例外が発生する
- 原因: `public`配下の未完成テンプレートにタイトルがなかった
- 解決方法: テンプレートを公開対象外の`templates/`へ移動する
- 制約: 実ブラウザの全操作ではなく、一覧APIとルートHTTP応答で修正を確認した
