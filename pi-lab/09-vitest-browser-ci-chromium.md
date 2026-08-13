# Vitest Browser ModeのCIにChromiumを導入する

## はじめに

同じエラーでも、原因を次の順で確認するとよい。

## 症状

Vitest Browser Modeを使うStorybookテストが、GitHub Actionsで次のエラーにより終了した。

```text
browserType.launch: Executable doesn't exist at
/home/runner/.cache/ms-playwright/chromium_headless_shell-1228/...

Looks like Playwright was just installed or updated.
Please run: npx playwright install
```

`@vitest/browser-playwright`がnpm依存に存在していても、Chromium実行ファイルとLinuxシステム依存がCIへ用意されているとは限らない。

## Browser Modeの構成

`vitest.config.ts`ではStorybook用プロジェクトがPlaywright providerとChromiumを指定していた。

```ts
browser: {
  enabled: true,
  headless: true,
  provider: playwright({}),
  instances: [
    {
      browser: "chromium",
    },
  ],
},
```

この設定は「どのブラウザで実行するか」を決めるが、そのブラウザをOSへインストールする処理ではない。

## CIのunit-test jobで導入する

Vitestを実行するjobへ、次のstepを追加した。

```yaml
- name: Install Playwright Chromium
  run: npx playwright install --with-deps chromium

- name: Vitest unit test
  run: npx vitest run
```

重要なのは、独立したPlaywright E2E jobではなく、実際にBrowser Modeを起動する`unit-test` jobへ置くことだ。GitHub Actionsのjob間では、同じrunnerやファイルシステムを前提にできない。

`--with-deps`はChromiumに必要なLinux依存も導入する。ブラウザを`chromium`へ限定することで、Vitestが使わないFirefoxやWebKitの取得を避けた。

## 切り分けの観点

同じエラーでも、原因を次の順で確認するとよい。

1. Browser Modeが指定するbrowser名
2. `@vitest/browser-playwright`とPlaywrightのnpm版
3. エラーに出たbrowser revision
4. そのjobで`playwright install`を実行しているか
5. cacheがnpm依存だけで、ブラウザ本体を含んでいない可能性

今回はログが`chromium_headless_shell-1228`の不在を明示していたため、テストコードやassertionではなく実行環境の問題と分類した。

## 検証と変更範囲

変更は`.github/workflows/frontend-ci.yaml`の3行だけで、コミットは`80241cf`である。アプリ実装やVitest assertionを弱めず、欠けていた実行依存をCIへ追加した。

## 制約

- Playwright更新でbrowser revisionも変わる
- jobやcontainerを分けた場合は、それぞれに実行ファイルが必要になる
- CJKを含むスクリーンショット比較では、ブラウザ依存とは別にフォント環境も揃える必要がある

## 参考資料

- [Vitest: Browser Mode](https://vitest.dev/guide/browser/)
- [Playwright: Continuous Integration](https://playwright.dev/docs/ci)
- 根拠コミット: `80241cf`
- 確認日: 2026-08-06

## まとめ

変更は`.github/workflows/frontend-ci.yaml`の3行だけで、コミットは`80241cf`である。アプリ実装やVitest assertionを弱めず、欠けていた実行依存をCIへ追加した。
