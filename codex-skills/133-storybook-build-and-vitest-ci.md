# Storybookの静的ビルドとVitestブラウザテストを独立してCIで実行する

## はじめに

提供されたフロントエンドテスト資料を、既存のStorybook 10.6/Vite構成へ適用しました。資料のjob分離という考え方を使い、テストには導入済みのVitest addonを利用しました。

## 前提・環境

2026年10月21日、Storybook 10.6.3、Vitest 5.0.5、Playwright Chromium、Bun 1.3.13です。資料は提供Drive内の `frontend_test_appendix.pdf`、Appendix A3、13〜20ページを参照しました。PDF本文や画像は転載していません。

## やってみた結果

Frontend CIへStorybook buildとStorybook testsを追加しました。両jobにneedsはありません。テストはソースをVitestで実行するため、ビルド済みStorybookのartifactをダウンロードしたり、静的サーバーを起動したりしません。

## 実装・検証

### 専用scriptを定義する

app/frontend/package.jsonへ次のscriptを追加しました。

```json
{
  "test:storybook": "vitest run --project=storybook",
  "test:storybook:coverage": "vitest run --project=storybook --coverage"
}
```

既存のvite.config.tsにはstorybookTestとPlaywright providerを使うstorybook projectがありました。専用scriptはそのprojectを選択します。

### ビルドとテストを別に確認する

app/frontendで実行しました。

```powershell
bun run build-storybook --quiet
bun run test:storybook:coverage --reporter=default --reporter=json --outputFile=coverage/test-result.json
```

静的ビルドと、3ファイル8テストが成功しました。ビルドartifactにはindex.html、iframe.html、8件のstory indexがありました。

| ローカルのStorybookカバレッジ指標 | 値 |
| --- | --- |
| Statements | 88.88% |
| Branches | 100% |
| Functions | 66.66% |
| Lines | 86.66% |

CIではChromiumとシステム依存をインストールし、静的出力は7日保持のartifactへ保存します。head `72e4776` の[push実行](https://github.com/kishimin/blog-wordcloud/actions/runs/39606078230)でもbuild/testsの成功を確認しました。

## つまずいたところ

資料は旧test-runnerによる静的ビルドの再利用を説明していました。一方、2026年10月21日に参照した[公式test-runner文書](https://storybook.js.org/docs/writing-tests/integrations/test-runner)は、Vite構成にVitest addonを推奨しています。既存構成へ追加のrunnerを持ち込まず、[Vitest addon](https://storybook.js.org/docs/writing-tests/integrations/vitest-addon)で実行する形を選びました。

## 学んだこと

参考資料では、実現したい検査と、当時のrunner・成果物の受け渡しを分けて読みました。今回のテストが消費しない静的artifactを、テストの前提にする必要はありませんでした。

## 根拠と制約

根拠は `6c6aaf2`、`ad502cc`、ローカル結果と上記CIです。8テストは例示コンポーネントを対象とし、main Appの欠落assetや型エラーを解決しません。VRTは追加しておらず、アクセシビリティ設定もtodoです。Functionsは80%未満で、品質目標をすべて達成した状態ではありません。
