# 手動で解析するmultipartフォームをSwagger UIに表示する

## 結論

Glyph Forgeの画像アップロードAPIは、リクエストをサイズ制限付きで手動解析している。そのため、`Request` を受け取る関数の引数だけではSwagger UIにファイル選択欄が現れなかった。`openapi_extra` でmultipartの入力項目とPNGの応答を明示し、Swagger UIから操作できる形にした。

## 発生した問題

`POST /images/frame-file` は画像と文字・色などを受け取り、画像を返す。実装は `Request` を受け、ボディサイズを制限してからフォームを解析する。OpenAPIにフォームの構造がなければ、Swagger UIはファイルや各項目の入力欄を組み立てられない。

## 対応

エンドポイントの `openapi_extra` に `multipart/form-data` のスキーマを追加した。`frame_image` は `type: string`, `format: binary` とし、画像と必須文字列を `required` に指定する。画像生成の設定は文字列や整数として記述する。実際のサイズ制限処理はエンドポイントに残した。

```python
"frame_image": {"type": "string", "format": "binary"},
"output_font_size": {
    "type": "integer",
    "default": DEFAULT_OUTPUT_FONT_SIZE,
    "minimum": MIN_OUTPUT_FONT_SIZE,
    "maximum": MAX_OUTPUT_FONT_SIZE,
},
```

数値入力の `default` には実装側の既定値を使った。応答も `image/png` のバイナリとして記述した。Swagger UIの操作画面と実際のAPIの両方を確認することが大切で、OpenAPIの記述だけでは実際のリクエスト処理まで保証できない。

## 動作確認と制約

- `tests/test_glyph_forge/test_app.py` にOpenAPIのフォーム項目、数値の既定値と範囲を検証するテストが追加されている。
- ローカルで起動した `glyph-forge:1.0.8` の `/openapi.json` を確認し、ファイルの `binary` 形式と `output_font_size` の既定値20、範囲10〜64を確認した。
- [PR #11](https://github.com/kishimin/glyph-forge/pull/11) の変更はマージ済み。対象の実装は [`0b55df2`](https://github.com/kishimin/glyph-forge/commit/0b55df2ab411e92d1ac3cf669a639a85a91eb64c)。
- 今回の確認はOpenAPIの定義を対象としており、この記録を書いた時点でSwagger UIの再操作は行っていない。

## 学んだこと

リクエストを直接扱う設計では、実行時の入力契約とOpenAPIの記述を別々に管理する必要がある。手動でスキーマを補う場合は、ファイル形式だけでなく、必須項目・既定値・制約・応答形式をテストし、実装とのずれを検出できるようにする。
