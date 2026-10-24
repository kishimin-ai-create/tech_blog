# WorkflowのYAML整形とActionsの静的検査を別々に確認する

## はじめに

CI設定を分割する際、YAMLの整形を再現できるコマンドが必要になりました。今回はPrettierの整形と、actionlintによるActions構文の確認を別の検証として行いました。

## 前提・環境

2026年10月21日、Bun 1.3.13、Prettier 3.9.10、actionlint 1.7.12を使ったblog-wordcloudでの記録です。

## やってみた結果

app/package.jsonへYAML専用のscriptを定義しました。

```json
{
  "format:yaml": "prettier --write \"../.github/workflows/*.{yml,yaml}\"",
  "format:yaml:check": "prettier --check \"../.github/workflows/*.{yml,yaml}\""
}
```

対象はリポジトリの.github/workflowsです。実行場所をappとし、そこからの相対パスをscriptへ含めています。

## 実装・検証

### ローカルで整形と検査を揃える

`bun run format:yaml` はファイルを書き換える操作です。確認だけの場合はappから `bun run format:yaml:check` を実行します。今回の3つのworkflowはcheckを通過しました。Prettierは固定版を宣言し、lockfileも同期しました。

### Actionsとしての問題を確認する

actionlintは公式リリースのWindows版を取得し、チェックサムを確認して.git配下のローカルツールとして実行しました。3つのworkflowは検査に合格しました。

actionlint自体をCI依存やアプリのpackage scriptへ追加したわけではありません。CIに常設したのはbackend check jobからの `format:yaml:check` です。

## つまずいたところ

YAMLが整形済みであることだけでは、jobの実行成功まで確認できませんでした。実際にYAML検査は成功していますが、ESLintによってbackend/frontendのcheck jobは失敗しています。

## 学んだこと

整形、Actions設定の静的検査、実行されたjobの結果には別の証拠が必要でした。ツールを実行した範囲と、CIへ組み込んだ範囲も分けて記録すると再現しやすくなります。

## 根拠と制約

根拠は[整形コマンド追加コミット](https://github.com/kishimin/blog-wordcloud/commit/2adfa5d)、app/package.json、backend-ci.yaml、actionlintとPrettierの実行結果です。formatter成功を、依存取得・テスト・権限まで含むworkflow全体の正常性とは扱いません。
