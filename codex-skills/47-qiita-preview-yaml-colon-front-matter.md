# Qiita Previewで記事APIが500になる：タイトル内のコロンをYAMLとして扱う

## 結論

Qiita Previewで特定の記事を開いたときにFront Matterの型エラーでHTTP 500になる場合、`title`などのYAML文字列に含まれるコロンを確認する。コロンの直後に空白がある値は、引用符で囲むと文字列として明確になる。

```yaml
title: "Pillow でフォント読み込み時に「OSError: cannot open resource」が発生した"
```

この記事では、2026年9月19日にQiita CLI 1.10.0で確認した事例を扱う。

## 発生した問題

Qiita Previewを起動した後、対象記事のAPIは次の項目を文字列・配列・真偽値として読めないという検証エラーを返した。

```text
titleは文字列で入力してください
tagsは配列で入力してください
privateの設定はtrue/falseで入力してください
```

記事ファイルの先頭は次のようになっていた。

```yaml
---
title: Pillow でフォント読み込み時に「OSError: cannot open resource」が発生した
tags:
  - Python
  - Pillow
private: false
---
```

## 調査

### 仮説1: ファイル先頭の区切りや文字コードが壊れている

先頭バイト、Front Matterの開始・終了行、および他の記事の先頭を比較した。開始行は`---`で、BOMもなく、区切りの形式は正常だった。

### 仮説2: タイトルの`OSError: cannot open resource`がYAMLの構文として曖昧

タイトル内には`OSError:`があり、コロンの後に空白が続いていた。これはYAMLのキーと値の区切りに使われる記法と重なる。タイトルを引用符で囲んだ後、Previewの対象記事APIはHTTP 200となり、検証エラーは0件になった。

## 原因

原因は、コロンと空白を含むタイトルを未クォートのYAML文字列として書いたことだった。Qiita CLIは解析結果を記事のFront Matterとして検証するため、期待したマッピングを得られないと、タイトルだけでなく他の必須項目も型エラーとして報告する。

## 解決方法

文字列全体を二重引用符で囲む。

```yaml
title: "Pillow でフォント読み込み時に「OSError: cannot open resource」が発生した"
```

この変更により、コロンはYAMLの構文ではなくタイトル本文の文字として扱われる。

## 動作確認

Preview起動中に、対象記事のAPIを確認した。

```powershell
Invoke-RestMethod -Uri 'http://localhost:8888/api/items/show?basename=newArticle14'
```

修正後はHTTP 200で応答し、`error_messages`は空だった。

## 再発防止

記事を追加した直後に、Previewで対象記事を開く。特にタイトル、説明、URLなどへコロンと空白を含める場合は、引用符で囲んでYAMLの値を明示する。

## まとめ

- 症状: Qiita Previewの対象記事APIがFront Matterの型エラーで500になる
- 原因: コロンと空白を含むタイトルが未クォートだった
- 解決方法: タイトルをYAML文字列として引用符で囲む
- 制約: この確認はQiita CLI 1.10.0のローカルPreviewで行った
