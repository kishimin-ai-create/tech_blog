# 壁時計に依存しない画面なら、E2EでPage Clockを固定しない

## はじめに

Playwrightの`page.clock`は、実際のタイマーを進めずに待機をスキップできる便利な機能だが、画面が壁時計（実際の日時）に依存する描画を持たないなら、E2Eテストでは使わない方がよい。クロックを操作すると、テストが検証する経路が「実際に2秒待って遷移する」という利用者体験ではなく、「クロックを進めて`setTimeout`をスキップする」というテスト専用の短絡経路にすり替わってしまう。Playwrightの`expect(locator).toBeVisible()`は既定で最大5秒ポーリングして待つため、数秒程度の実待機ならクロック操作なしでも決定的にテストできる。

## 結論

Playwrightの`page.clock`は、実際のタイマーを進めずに待機をスキップできる便利な機能だが、画面が壁時計（実際の日時）に依存する描画を持たないなら、E2Eテストでは使わない方がよい。クロックを操作すると、テストが検証する経路が「実際に2秒待って遷移する」という利用者体験ではなく、「クロックを進めて`setTimeout`をスキップする」というテスト専用の短絡経路にすり替わってしまう。Playwrightの`expect(locator).toBeVisible()`は既定で最大5秒ポーリングして待つため、数秒程度の実待機ならクロック操作なしでも決定的にテストできる。

## 変更前の構成

pi-labの`src/features/pi-message/pages/pi-message.tsx`は、メッセージ送信後に2秒待ってから結果画面へ切り替わる。

```tsx
useEffect(() => {
  if (pageType === "progress") {
    const timer = setTimeout(() => {
      setPageType("result");
    }, 2000);

    return () => {
      clearTimeout(timer);
    };
  }
}, [pageType]);
```

この2秒を実際に待たずにテストするため、`e2e/specs/pi-message.spec.ts`と`e2e/specs/vrt.spec.ts`は`page.clock`でブラウザの時刻を操作していた。

```ts
await page.clock.install();
await piMessagePage.goto();
// ...
await page.clock.pauseAt(Date.now() + 60_000);
await piMessagePage.messageInput.submitMessage();
// ...
await page.clock.fastForward(2_000);
await expect(piMessagePage.messageResult.getMessageResult).toBeVisible();
```

## 指摘

レビューで次の指摘があった。

> モックはやめてほしいね。E2Eなのに

E2E（End-to-End）テストの狙いは、実際の利用者が体験する経路をできる限りそのまま検証することにある。クロックを操作して`setTimeout`を迂回すると、検証しているのは「2秒後に結果画面が出る」という実際の挙動ではなく、「クロックを進めればすぐ切り替わる」という実装の抜け道になってしまう。

## 原因調査：本当にクロック固定が必要か

`src/`配下を`Date.now()`・`new Date()`・`toLocaleString`・`toLocaleTimeString`でgrepしたところ、該当箇所は1件もなかった。つまりこのアプリには、画面に表示される内容が実行時刻（壁時計）によって変わる箇所が存在しない。

`page.clock`が実際に行っていたのは、時刻の値そのものを固定することではなく、「2秒の`setTimeout`をスキップする」という待機時間の短縮だけだった。Playwrightの`expect(locator).toBeVisible()`は既定で最大5秒までポーリングして待つ仕様のため、2秒程度の実待機はクロック操作なしでも自動的に吸収できる。

## 対応

`pi-message.spec.ts`と`vrt.spec.ts`の両方から`page.clock.install()`・`pauseAt()`・`fastForward()`を削除した。

```ts
// Before
await page.clock.pauseAt(Date.now() + 60_000);
await piMessagePage.messageInput.submitMessage();
await expect(
  piMessagePage.progressingMessage.getProgressMessage,
).toBeVisible();
// ...
await page.clock.fastForward(2_000);
await expect(piMessagePage.messageResult.getMessageResult).toBeVisible();
```

```ts
// After
await piMessagePage.messageInput.submitMessage();
await expect(
  piMessagePage.progressingMessage.getProgressMessage,
).toBeVisible();
// ...
await expect(piMessagePage.messageResult.getMessageResult).toBeVisible();
```

`vrt.spec.ts`の`Math.random`モック（`page.addInitScript`）はこの変更の対象外とした。こちらは画面に表示される乱数依存の文言をスクリーンショット比較のために固定するためのもので、時刻操作とは目的が異なる。

## やってみた結果

削除後、両specともPlaywrightで再実行しGreenを確認した。

```text
$ npx playwright test e2e/specs/pi-message.spec.ts --project=chromium
1 passed (4.1s)

$ npx playwright test e2e/specs/vrt.spec.ts --project=chromium
（クロック削除前後で同一の、無関係な既存差分1件を除き成功）
```

修正コミットは`8cec845`（pi-message.spec.ts）と`5b2a58a`（vrt.spec.ts）である。

## 学んだこと

- `page.clock`のようなタイマー操作APIは、「画面が壁時計に依存する描画を持つ」場合と「相対的なタイマーで画面遷移するだけ」の場合とで、必要性が全く異なる。前者には引き続き必要だが、後者ではPlaywrightの自動待機で十分な場合が多い。
- 使う前に、対象コードに`Date.now()`・`new Date()`・`toLocaleString`系の呼び出しがあるかをgrepで確認する。なければクロック固定は不要な可能性が高い。
- E2Eテストで「本当に実際の経路を通っているか」を疑う視点は、モックや時間操作を導入するたびに持つ価値がある。テストがGreenになることと、実際の利用者体験を検証できていることは別の問題である。

## 参考資料

- [Playwright: Clock](https://playwright.dev/docs/clock)
- [Playwright: Auto-waiting](https://playwright.dev/docs/actionability)
- 根拠コミット: `8cec845`, `5b2a58a`
- 確認日: 2026-08-07

## まとめ

修正コミットは`8cec845`（pi-message.spec.ts）と`5b2a58a`（vrt.spec.ts）である。
