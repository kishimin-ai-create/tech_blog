# Playwrightでダウンロード成果物をテスト後も確認できるようにする

## はじめに

Playwrightの`Download`イベントを待つだけでは、テスト終了後に生成ファイルを確認できません。この記事では、リリースE2Eで生成したPNGをテスト成果物として保存する方法を記録します。

## 前提・環境

- Runtime: Bun
- Framework: Playwright Test
- 対象: `frontend/e2e/tests/image-generation.large.test.ts`

## やってみた結果

`Download.saveAs()`へ`testInfo.outputPath()`を渡すことで、各テストの出力ディレクトリに生成PNGを保存できるようになりました。スクリーンショットとは別名で保存するため、両方の証拠を確認できます。

## 実装・検証

### Step 1: ダウンロードを待つ

画像生成操作から`Download`を受け取ります。ここではブラウザのダウンロードイベントが発生したことと、ファイル名の形式を検証します。

```ts
const download = await imageGenerationPage.submit();

expect(download.suggestedFilename()).toMatch(
  new RegExp(`^mojica-${imageType}-[0-9a-f-]+\\.png$`),
);
```

### Step 2: テスト成果物として保存する

`testInfo.outputPath()`は、テストごとの成果物ディレクトリ内の安全なパスを作ります。そのパスを`saveAs`へ渡します。

```ts
await download.saveAs(testInfo.outputPath("ja-standard-download.png"));
```

同じテストのスクリーンショットは別名で保存します。

```ts
await page.screenshot({
  path: testInfo.outputPath("ja-standard.png"),
  fullPage: true,
});
```

検証では、対象ファイルのPrettier、TypeScript、Oxlint、ESLintを実行し、Playwrightの`--list`で30テストが列挙されることを確認しました。

## つまずいたところ

### 問題

`Download`を受け取ってファイル名だけを検証していたため、生成されたPNGをテスト後に確認できませんでした。

### 原因

Playwrightが管理するダウンロードは、一時ファイルとして扱われます。永続的な確認用ファイルが必要な場合は、テスト自身が保存する必要があります。

### 解決

`download.saveAs(testInfo.outputPath(...))`を呼び出し、スクリーンショットと同じテスト成果物領域へ保存しました。

## 学んだこと

`Download`イベントの成功確認と、ファイル内容を後から調査できる成果物保存は別の責務です。リリーステストでは、ファイル名のアサートに加えて保存先を明示すると、失敗時の調査材料を残せます。

## まとめ

- `Download`はイベントを待つだけでは永続保存されない
- `saveAs`と`testInfo.outputPath`でテスト成果物として保存できる
- ダウンロードとスクリーンショットは別名にして同時に確認できる

## 参考資料

- [Playwright Download API](https://playwright.dev/docs/api/class-download)
- [Playwright TestInfo API](https://playwright.dev/docs/api/class-testinfo)
