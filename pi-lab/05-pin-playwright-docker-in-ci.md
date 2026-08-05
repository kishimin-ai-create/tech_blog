# PlaywrightのVRT環境をDockerで固定する

## 結論

VRTの基準画像をDockerで生成したなら、CIも同じPlaywright Dockerイメージで実行する。OS名がどちらもLinuxでも、ホストイメージ、フォント、システムライブラリが違えば画像差分は発生する。

pi-labでは`ubuntu-latest`へブラウザを後からインストールする方式から、Playwright 1.61.1のUbuntu Nobleイメージを使うcontainer jobへ変更した。

## 変更前の構成

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: lts/*
- run: npm ci
- run: npx playwright install --with-deps
- run: npx playwright test
```

この構成は一般的なE2Eには有効だが、ローカルで基準画像を生成したDocker環境とは一致しなかった。実際に、CIのActual画像だけ文字が太く描画され、1,436ピクセルの差分が安定して発生した。

## 固定イメージを使う

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    container:
      image: mcr.microsoft.com/playwright:v1.61.1-noble
      options: --init --ipc=host
    steps:
      - uses: actions/checkout@v4
      - name: Install dependencies
        run: npm ci
      - name: Install Microsoft Edge
        run: npx playwright install msedge
      - name: Run Playwright tests
        run: npx playwright test
```

公式イメージにはブラウザとブラウザ用システム依存が含まれるが、プロジェクトのnpm依存は含まれないため`npm ci`は必要である。

## Dockerイメージとnpm版を一致させる

プロジェクトのlockfileが解決した`@playwright/test`は1.61.1だった。このためDockerタグも`v1.61.1-noble`へ固定した。

Playwright公式ドキュメントは、Dockerイメージとテスト側のPlaywright版が一致しないと、期待するブラウザ実行ファイルを見つけられない場合があると説明している。`latest`相当の可変タグではなく、具体的な版を使う。

## Microsoft Edgeは追加で導入する

公式イメージ内を確認したところ、Chromium、Firefox、WebKitは存在したがMicrosoft Edgeはなかった。既存の`Microsoft Edge`プロジェクトを失わないため、Edgeだけを追加インストールした。

VRTだけを通すためにEdgeプロジェクトを削除すると、テスト範囲が変わってしまう。環境固定とブラウザ範囲維持を別の要件として扱うことが重要だった。

## 検証結果

固定イメージ内でEdgeをインストールし、CI相当の全E2Eを実行した。

```text
Running 20 tests using 1 worker
20 passed
```

型検査も成功し、VRTは更新なしで全基準画像と一致した。変更コミットは`e261542`である。

## トレードオフ

- Edgeの追加インストール時間が増える
- Playwrightを更新するときはnpm lockfileとDockerタグを同時に更新する必要がある
- 固定環境により再現性は上がるが、別OS固有の問題は別ジョブで検証する必要がある

## 参考資料

- [Playwright: Docker](https://playwright.dev/docs/docker)
- [Playwright: Continuous Integration](https://playwright.dev/docs/ci)
- 根拠コミット: `e261542`
- 確認日: 2026-08-06
