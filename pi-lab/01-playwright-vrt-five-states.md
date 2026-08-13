# Playwrightで主要導線の5状態を画像比較する

## 結論

スクリーンショットをレポートへ添付するだけでは、見た目が変わってもテストは失敗しない。視覚的な回帰を自動検出したい場合は、Playwrightの`toHaveScreenshot()`を合否条件にする。

この記事は、機能E2Eはあるものの、レイアウト、色、寸法、配置の変化を自動検出できていないプロジェクトを対象にする。

## 対象にした利用者導線

pi-labでは、次の5状態を1つの利用者導線として比較した。

1. アプリ一覧
2. メッセージ入力済み
3. 計算処理中
4. 計算結果
5. 再試行後の空入力

単に5枚撮影するのではなく、各撮影の前にURL、表示要素、入力値をweb-first assertionで確認している。画像だけに依存すると、「目的の状態へ到達していない画面」を誤って基準画像にする危険があるためだ。

## 比較処理を小さくまとめる

`e2e/specs/vrt.spec.ts`では、比較条件をヘルパーへ集約した。

```ts
const compareScreenshot = async (page: Page, name: string) => {
  await expect(page).toHaveScreenshot(name, {
    animations: "disabled",
    fullPage: true,
  });
};
```

`fullPage: true`は画面全体を比較対象にする。`animations: "disabled"`は、CSSアニメーションの途中フレームが差分になることを避ける。ただし、時間や乱数まで自動で固定されるわけではない。これらはテスト側で別途制御する。

## 状態確認後に撮影する

入力状態なら、値が反映されたことを確認してから比較する。

```ts
await piMessagePage.messageInput.fillMessageInput("割り切れない研究所");
await expect(piMessagePage.messageInput.getMessageInput).toHaveValue(
  "割り切れない研究所",
);
await compareScreenshot(page, "02-pi-message-input.png");
```

結果状態でも同様に、結果と再試行ボタンの表示を先に確認する。

```ts
await expect(piMessagePage.messageResult.getMessageResult).toBeVisible();
await expect(piMessagePage.messageResult.getRetryButton).toBeVisible();
await compareScreenshot(page, "04-pi-message-result.png");
```

これにより、機能上の契約はロケーターで、視覚上の契約は画像で表現できる。

## 初回生成と通常比較を分ける

意図した画面を確認したうえで基準画像を生成する。

```bash
npx playwright test e2e/specs/vrt.spec.ts --update-snapshots
```

通常の検証では更新フラグを付けない。

```bash
npx playwright test e2e/specs/vrt.spec.ts
```

差分が出たときに無条件で`--update-snapshots`を実行すると、本物の視覚的回帰まで正解として上書きしてしまう。Expected、Actual、Diffを確認してから更新する。

## 比較結果

コミット`b0d093d`で画像添付から画像比較へ移行し、`3bf6d6c`でWindows用の5枚を登録した。その後、対象ブラウザを拡張した最終構成では、固定Linux環境でVRT 5件、全E2E 20件が成功した。

## トレードオフ

ブラウザの描画はOS、ブラウザ版、フォント、headless設定などで変わる。同じ基準画像を異なる環境へ無理に共有せず、生成環境と比較環境を揃える必要がある。

## 参考資料

- [Playwright: Visual comparisons](https://playwright.dev/docs/test-snapshots)
- 根拠コミット: `b0d093d`、`3bf6d6c`
- 確認日: 2026-08-06

## まとめ

ブラウザの描画はOS、ブラウザ版、フォント、headless設定などで変わる。同じ基準画像を異なる環境へ無理に共有せず、生成環境と比較環境を揃える必要がある。
