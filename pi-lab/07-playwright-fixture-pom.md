# Playwright Fixtureの上書きでPOMを配布する

## 結論

複数のE2E specでPage Object Model（POM）の生成と初期遷移を繰り返すなら、Playwrightのtyped fixtureへ集約できる。組み込み`page` fixtureを上書きすれば、全テストの開始地点も1か所で保証できる。

pi-labでは`e2e/fixtures/test.ts`を作り、`PiLoopPage`と`PiMessagePage`をテスト引数として配布した。

## 変更前の重複

各specがPOMを直接生成し、必要に応じて`goto()`を呼んでいた。

```ts
const piLoopPage = new PiLoopPage(page);
const piMessagePage = new PiMessagePage(page);

await piLoopPage.goto();
```

この形では、テストごとに初期化方法がずれやすく、POM追加時にも複数ファイルを編集する必要がある。

## typed fixtureを定義する

```ts
import { test as base } from "playwright/test";
import { PiLoopPage } from "../pages/pi-loop-page";
import { PiMessagePage } from "../pages/pi-message-page";

type PageObjectFixtures = {
  piLoopPage: PiLoopPage;
  piMessagePage: PiMessagePage;
};

export const test = base.extend<PageObjectFixtures>({
  piLoopPage: async ({ page }, provide) => {
    await provide(new PiLoopPage(page));
  },
  piMessagePage: async ({ page }, provide) => {
    await provide(new PiMessagePage(page));
  },
});

export { expect } from "playwright/test";
```

fixture名と型を対応付けるため、spec側では補完と型検査を維持できる。Playwrightはテストが要求したfixtureだけを準備する。

## 組み込みpage fixtureを上書きする

pi-labでは、すべてのテストを`baseURL`から開始させるため`page`も上書きした。

```ts
page: async ({ baseURL, page }, provide) => {
  if (baseURL === undefined) {
    throw new Error("Playwright baseURL must be configured");
  }

  await page.goto(baseURL);
  await provide(page);
},
```

`baseURL`がない場合に空文字や推測値へフォールバックせず、構成エラーとして即座に失敗させている。これにより、相対URLを使うテストが偶然別のページで動くことを避けられる。

## spec側は利用者の操作へ集中する

```ts
import { expect, test } from "../fixtures/test";

test("πで伝える画面へ遷移する", async ({
  page,
  piLoopPage,
  piMessagePage,
}) => {
  await piLoopPage.gotoPiMessage();
  await expect(page).toHaveURL("/pi-message");
  await expect(piMessagePage.getPageTitle).toBeVisible();
});
```

POMの生成コードが消え、テストには操作と期待結果が残る。fixtureはセットアップ境界、POMは画面操作、specは振る舞いの記述という役割分担になる。

## 注意点

- fixture内へ業務上のassertionを詰め込まない
- 自動遷移が不要なテストまで同じfixtureへ強制しない
- `provide()`より前をsetup、後ろをteardownとして扱う
- worker scopeが必要な高コスト資源と、テストごとに分離すべきPageを混同しない

コミット`9fc5ee4`では3つのspecからPOMの重複生成を削除し、fixtureへ一本化した。

## 参考資料

- [Playwright: Fixtures](https://playwright.dev/docs/test-fixtures)
- [Playwright: Overriding fixtures](https://playwright.dev/docs/test-fixtures#overriding-fixtures)
- 根拠コミット: `9fc5ee4`
- 確認日: 2026-08-06

## まとめ

コミット`9fc5ee4`では3つのspecからPOMの重複生成を削除し、fixtureへ一本化した。
