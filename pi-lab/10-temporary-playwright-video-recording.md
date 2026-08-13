# Playwrightの動画撮影設定を一時利用に限定する

## はじめに

`slowMo`はPlaywright操作を遅くする設定であり、アプリ内タイマーを制御する機能ではない。処理中画面から2秒後に結果へ遷移するシナリオでは、Clockも併用した。

## 結論

画面遷移のレビュー用動画を一度だけ必要とする場合、恒久的なPlaywright設定として残さない。専用プロジェクトで撮影条件を限定し、動画取得後は設定を撤去して通常のブラウザ行列へ戻す。

pi-labではGoogle Chromeだけで動画を撮影し、操作を見やすくするため`slowMo: 2_000`を一時設定した。その後、コミット`8f9c266`で動画設定を削除した。

## 一時的に使った専用プロジェクト

```ts
{
  name: "Google Chrome VRT",
  testMatch: "**/vrt.spec.ts",
  use: {
    ...devices["Desktop Chrome"],
    channel: "chrome",
    video: "on",
    launchOptions: { slowMo: 2_000 },
  },
},
```

既存プロジェクト側では`testIgnore: "**/vrt.spec.ts"`を一時的に設定し、同じVRTが複数ブラウザで重複撮影されないようにした。

`channel: "chrome"`はPlaywright同梱Chromiumではなく、Google Chrome channelを選ぶ。`slowMo`は各操作を2秒遅らせ、レビュー動画で操作の前後を追いやすくするための撮影条件だった。

## 時間依存UIはslowMoだけでは安定しない

`slowMo`はPlaywright操作を遅くする設定であり、アプリ内タイマーを制御する機能ではない。処理中画面から2秒後に結果へ遷移するシナリオでは、Clockも併用した。

```ts
await page.clock.install();
await page.clock.pauseAt(Date.now() + 60_000);
await piMessagePage.messageInput.submitMessage();
await expect(progressMessage).toBeVisible();
await page.clock.fastForward(2_000);
await expect(resultMessage).toBeVisible();
```

撮影速度とアプリ時刻を別々に扱うことで、動画の見やすさとテストの決定性を両立できる。

## 動画はテスト終了後に確定する

Playwrightの動画はBrowserContextが閉じられた時点で保存される。テスト途中でパスを参照しても、ファイルがまだ完成していない場合がある。通常はテスト終了後、`test-results`配下の成果物を確認する。

## なぜ恒久設定にしなかったか

`video: "on"`と`slowMo: 2_000`を残すと、日常のE2Eでも次のコストが発生する。

- 成功テストを含む動画生成
- テスト時間の増加
- `test-results`の容量増加
- VRTだけ特別なChromeプロジェクトへ閉じ込める設定複雑性

動画取得後に専用プロジェクトと`testIgnore`を削除し、元のChromium、WebKit、モバイル、Edge構成へ戻した。恒常的な障害解析が目的なら、`video: "retain-on-failure"`または`"on-first-retry"`を検討できるが、今回の一時レビューとは目的が異なる。

## 変更履歴

- `9569c77`: 主要導線の5状態を記録するspecを追加
- `be940de`: Google Chrome、動画、`slowMo: 2_000`を一時設定
- `8f9c266`: 動画取得後に一時設定を撤去

## 参考資料

- [Playwright: Videos](https://playwright.dev/docs/videos)
- [Playwright: Clock](https://playwright.dev/docs/clock)
- 確認日: 2026-08-06

## まとめ

動画取得後に専用プロジェクトと`testIgnore`を削除し、元のChromium、WebKit、モバイル、Edge構成へ戻した。恒常的な障害解析が目的なら、`video: "retain-on-failure"`または`"on-first-retry"`を検討できるが、今回の一時レビューとは目的が異なる。
