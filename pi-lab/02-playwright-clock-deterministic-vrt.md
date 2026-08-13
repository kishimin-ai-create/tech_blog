# Playwright Clockで時間依存VRTを決定的にする

## はじめに

pi-labでは、メッセージ送信後に処理中画面を表示し、2秒後に結果へ遷移する。この状態を高速かつ再現可能に検証するため、`page.clock`を使用した。

## 結論

時間経過で表示が変わる画面をVRTする場合、実時間の待機ではなくPlaywright Clockで時間を進める。乱数も画面へ影響するなら、ページのスクリプトが動く前に固定する。

pi-labでは、メッセージ送信後に処理中画面を表示し、2秒後に結果へ遷移する。この状態を高速かつ再現可能に検証するため、`page.clock`を使用した。

## Clockはページ読み込みより前に導入する

```ts
await page.clock.install();
await page.addInitScript(() => {
  Math.random = () => 0.5;
});
await piLoopPage.goto();
```

Playwright公式ドキュメントは、`install()`を使う場合、対象となるClock APIがページで使われる前に呼び出す必要があると説明している。ページ遷移後に導入すると、アプリが既に保持したタイマーとテストが制御するタイマーが分かれる可能性がある。

`addInitScript()`もページスクリプトより先に実行される。これにより、計算結果へ影響する`Math.random()`を毎回`0.5`へ固定した。

## 現在時刻へ戻さない

当初は次のコードを使っていた。

```ts
await page.clock.pauseAt(new Date());
```

しかし、`new Date()`が返すホスト時刻と、既に進行したブラウザ内Clockの時刻がずれると、Clockを過去へ戻す操作になり得る。並列ブラウザ実行ではセットアップ時間の差も加わる。

修正後は現在値より明確に未来を指定した。

```ts
await page.clock.pauseAt(Date.now() + 60_000);
```

この変更はコミット`be07691`で行った。目的は60秒待つことではなく、Clockの停止地点が現在より未来であることを保証する点にある。

## 実時間を待たずに結果へ進める

```ts
await piMessagePage.messageInput.submitMessage();
await expect(
  piMessagePage.progressingMessage.getProgressMessage,
).toBeVisible();

await page.clock.fastForward(2_000);
await expect(piMessagePage.messageResult.getMessageResult).toBeVisible();
```

固定の`waitForTimeout(2000)`では、CI負荷による遅延と実時間コストが残る。`fastForward()`なら、対象タイマーを進めつつテスト時間を短縮できる。

## 決定性の確認方法

次の順で確認する。

1. 処理中状態が表示されること
2. 処理中状態の画像が基準画像と一致すること
3. Clockを2秒進めること
4. 結果状態が表示されること
5. 結果状態の画像が基準画像と一致すること

最終構成では、Chromium、WebKit、Mobile Chrome、Mobile Safari、Microsoft Edgeの全5プロジェクトでこのシナリオが更新なしの画像比較に成功した。

## 制約

Clockはブラウザ内の時間関連APIを置き換える。サーバー時刻、外部API、別プロセスのジョブは同じ方法では制御できない。また、日時文字列そのものを基準画像へ含める場合は、`install({ time })`や`setFixedTime()`で絶対時刻まで固定する必要がある。

## 参考資料

- [Playwright: Clock](https://playwright.dev/docs/clock)
- 根拠コミット: `be07691`
- 確認日: 2026-08-06

## まとめ

Clockはブラウザ内の時間関連APIを置き換える。サーバー時刻、外部API、別プロセスのジョブは同じ方法では制御できない。また、日時文字列そのものを基準画像へ含める場合は、`install({ time })`や`setFixedTime()`で絶対時刻まで固定する必要がある。
