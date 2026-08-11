# テスト設定を触る前に、公式ドキュメントで既定値と用法を確認する

## 結論

「このオプションは本当に必要か」「このやり方は公式に推奨されているか」を判断するとき、記憶や慣習ではなくPlaywright公式ドキュメントを都度確認する。pi-labでは、この確認によって(1)既定値と同じ値を明示していただけの冗長なオプションを1つ削除でき、(2)一見モックに見えるコードが公式ドキュメントの想定用法と一致することを確認できた。

## ケース1：既定値を明示していただけのオプション

`e2e/specs/vrt.spec.ts`の`compareScreenshot`ヘルパーは、スクリーンショット比較のたびに`animations: "disabled"`を明示的に渡していた。

```ts
const compareScreenshot = async (page: Page, name: string) => {
  await expect(page).toHaveScreenshot(name, {
    animations: "disabled",
    fullPage: true,
  });
};
```

レビューで次のやり取りがあった。

> これは公式にも載ってるかね？

Playwright公式ドキュメント（`toHaveScreenshot`の`animations`オプション）を確認したところ、`"disabled"`は指定可能な値の1つであると同時に、**既定値そのもの**であることが分かった。

> デフォルトならいらないのでは？

その通りだったため、明示指定を削除した。

```ts
const compareScreenshot = async (page: Page, name: string) => {
  await expect(page).toHaveScreenshot(name, {
    fullPage: true,
  });
};
```

`playwright.config.ts`にも`expect.toHaveScreenshot`のオーバーライドがないことを確認済みであり、既定値がそのまま使われる。挙動は変わらず、コードから「なぜこの値を指定しているのか」を読者が推測する必要がなくなった。

## ケース2：一見モックに見えるコードが公式の想定用法だったケース

同じ`vrt.spec.ts`には、次のように`Math.random`を上書きするコードもある。

```ts
await page.addInitScript(() => {
  Math.random = () => 0.5;
});
```

E2Eテストでの値のモックはケースバイケースで是非が分かれるため、`page.addInitScript`自体がどう位置づけられているAPIかをPlaywright公式ドキュメント（`page.addInitScript()`）で確認した。

ドキュメントには、このAPIが「ドキュメント生成後、ページ自身のスクリプトが実行される前」に評価されるスクリプトを追加するものであり、代表例として**まさに`Math.random`の上書き**が挙げられていた。「アプリのコードが動き出す前に確定的な乱数へ差し替える」というのは、公式が想定する典型的なユースケースそのものである。

この確認により、`Math.random`のモックは[別記事](./14-stop-mocking-clock-without-wall-clock-rendering.md)で削除した`page.clock`とは性質が異なる——公式が推奨する用法に沿った、画面の表示内容（乱数依存の文言）を決定的にするための処置である——と判断でき、削除せず維持する判断に根拠を持たせられた。

## 検証結果

`animations`オプション削除後、`npx eslint`・`npx tsc --noEmit`はエラーなし。`npx playwright test e2e/specs/vrt.spec.ts --project=chromium`は、既存の無関係な差分1件を除きスクリーンショット比較の結果自体に変化がないことを確認した（既定値を明示から省略しただけなので当然ではあるが、実際に確認した）。

## 学び

- 「これは公式にあるか」「これは既定値ではないか」という問いは、コードを削る方向にも残す方向にも使える。ケース1では削除の根拠になり、ケース2では維持の根拠になった。
- 公式ドキュメントを確認するコストは小さく、確認せずに「たぶんこうだろう」で判断すると、既定値の重複のような無害だが無意味なコードが後から增えていく。
- 特にテストのオプション設定は、書いた本人以外が「なぜこの値なのか」を後から読み解くのが難しい。既定値と同じなら消す、既定値と違うなら理由をコメントかコミットメッセージに残す、のどちらかにする。

## 参考資料

- [Playwright: toHaveScreenshot（`animations`オプション）](https://playwright.dev/docs/api/class-pageassertions#page-assertions-to-have-screenshot-1)
- [Playwright: page.addInitScript()](https://playwright.dev/docs/api/class-page#page-add-init-script)
- 根拠コミット: `5b2a58a`
- 確認日: 2026-08-07
