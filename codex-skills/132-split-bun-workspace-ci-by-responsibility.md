# Bun workspaceのCIを分けてもインストールはworkspaceルートで行う

## はじめに

一つのbuild-app workflowをfrontend・backend・schemasへ分けました。検査の責任はパッケージごとに分け、依存インストールは共通lockfileを持つappから実行しています。

## 前提・環境

2026年10月21日のblog-wordcloudです。GitHub Actionsはubuntu-24.04、Node 22、Bun 1.3.13を指定しています。

## やってみた結果

各workflowをpushとpull_requestで起動し、pathフィルターは設けませんでした。backendの静的検査とテスト、frontendの静的検査・Storybookビルド・Storybookテストを独立したjobへ分けました。

## 実装・検証

### インストールと実行の場所を分ける

各jobはcheckout後に `app` で次を実行します。

```shell
bun install --frozen-lockfile
```

その後、対象パッケージのscriptを呼びます。frontend jobの既定ディレクトリはapp/frontendですが、install stepだけappを明示します。backend・schemas側はappから `bun run --cwd <package> ...` を呼びます。

### 静的検査の失敗とテスト結果を別に残す

check jobが失敗しても、別jobのテストは実行できます。同じcheck job内の後続検査には、install成功かつ非cancelを条件に指定しました。型エラーが出てもESLintの結果を取得するためです。

frontendは実際のVitest JSON結果を生成し、件数が0または結果ファイルがなければレポート処理を失敗させます。PRへのコメントは同一リポジトリからのPRで、Dependabot以外の場合に限定します。forkやpushではsummaryとartifactを残します。

イベントと結果を組み合わせた8ケースをモックで確認しました。これはfork PR上の権限を実測した証拠ではありません。

## 観測結果

最新head `72e4776` のpush実行では、Schemas checks、Backend tests、Storybook build、Storybook testsが成功しました。Frontend checksは型チェックとESLint、Backend checksはESLintで失敗しています。

[Frontend CIの実行](https://github.com/kishimin/blog-wordcloud/actions/runs/39606078230)と[Backend CIの実行](https://github.com/kishimin/blog-wordcloud/actions/runs/39606078223)でjobごとの結果を確認しました。Linux側のbackend Biomeは成功し、Windows側のCRLF整形失敗とは結果が異なりました。

## 学んだこと

CIを分けた効果は、成功と失敗を対象ごとに観測できたことです。待ち時間の短縮量は計測していません。また、テスト成功から静的解析やアプリ全体の正常動作を推定することも避けました。

## 根拠と制約

根拠は `9e2b9bc`、`45d2097`、`3ab3bc7`、`ad502cc` と3つのworkflow、上記の実行結果です。schemasには専用テストscriptがなく、CIで型検査とBiomeだけを実行します。全CI成功やカバレッジ目標達成は未完了です。
