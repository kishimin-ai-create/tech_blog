# Playwright Artifactから画像差分の原因を調べる

## 結論

CIのVRT失敗は、ログの差分ピクセル数だけで原因を決めない。GitHub Actionsの`playwright-report` Artifactを取得し、Expected、Actual、Diffを並べて、差分が文字、配置、色面、画像のどこに集中しているかを確認する。

## 解析した症状

pi-labでは次の失敗を解析した。

```text
1436 pixels (ratio 0.01 of all image pixels) are different.

Expected: 01-pi-loop-chromium-linux.png
Received: 01-pi-loop-actual.png
Diff:     01-pi-loop-diff.png
```

対象画像は1280×720、総ピクセル数は921,600だった。1,436ピクセルは約0.156%に相当する。Playwright 1.61.1のログに出た`ratio 0.01`だけを見て「画像全体の1%が違う」と断定せず、正確なピクセル数と画像寸法も確認する必要がある。

## Artifactを取得する

GitHub Actionsの実行ページ下部にあるArtifactsから`playwright-report`をダウンロードする。Artifactは、ワークフロー実行後に別ジョブや手元の解析で使うファイルを保存する仕組みである。

ZIPを展開後、HTMLレポートを開く。

```powershell
npx playwright show-report C:\path\to\playwright-report
```

失敗したテストのAttachmentsで、次の3枚を切り替える。

- Expected: リポジトリの基準画像
- Actual: CIで撮影した画像
- Diff: 差分位置を強調した画像

## 今回の観測結果

Diff画像では、背景色やレイアウト全体ではなく、見出し、リンク、フッター、ロゴの輪郭へ差分が集中していた。Actualの文字はExpectedより太く描画されていた。また、リトライ3回でActualとDiffの内容ハッシュが同じだった。

この2点から、アニメーションによる一時的な揺れではなく、実行環境のフォントまたは描画依存差と分類した。これは画像から確認した事実に基づく分類であり、単に「Linuxだから」と推測したものではない。

## 安易に許容差を増やさない

`maxDiffPixels`を1,436以上にすれば、その時点の失敗は通せる。しかし文字全体の描画が変わったケースを許容すると、将来の意図しないfont-weight変更まで見逃しやすくなる。

今回採用した対処は、基準画像生成環境とCIを同じPlaywright Dockerイメージへ固定することだった。修正後、CI相当環境の全E2Eは20件すべて成功した。

## Traceも併用する

操作やDOM状態が疑わしい場合は、HTMLレポート内のTraceを確認する。PlaywrightはCIで`trace: "on-first-retry"`を使う構成を案内しており、操作、DOM snapshot、ネットワーク、コンソールを時系列で調べられる。

## 制約

HTMLレポートだけで、使用された実フォント名やOSパッケージを常に確定できるわけではない。必要ならTrace、ブラウザログ、`document.fonts`、コンテナ内のフォント一覧を追加で採取する。

## 参考資料

- [GitHub Docs: Store and share data with workflow artifacts](https://docs.github.com/en/actions/tutorials/store-and-share-data)
- [Playwright: Trace viewer](https://playwright.dev/docs/trace-viewer)
- [Playwright: Visual comparisons](https://playwright.dev/docs/test-snapshots)
- 確認日: 2026-08-06
