# Page ObjectのLocatorは、増える前提なら引数付きメソッドにする

## はじめに

Page Objectに「一覧の中の特定の1項目」を指すLocatorを固定プロパティとして持たせると、一覧の項目が増えるたびにプロパティとメソッドが増殖する。項目を特定する値（リンク名など）を引数に取るメソッドへ最初から設計しておけば、項目が増えても変更不要になる。

## 結論

Page Objectに「一覧の中の特定の1項目」を指すLocatorを固定プロパティとして持たせると、一覧の項目が増えるたびにプロパティとメソッドが増殖する。項目を特定する値（リンク名など）を引数に取るメソッドへ最初から設計しておけば、項目が増えても変更不要になる。

## 変更前の構成

`e2e/pages/pi-loop-page.ts`は、一覧内の特定のリンクだけを指す固定プロパティを持っていた。

```ts
export class PiLoopPage extends BasePage {
  readonly getList: Locator;
  readonly getPiMessageLink: Locator;

  constructor(page: Page) {
    super(page);
    this.getList = page.getByRole("list");
    this.getPiMessageLink = this.getList
      .getByRole("listitem")
      .getByRole("link", { name: /πで伝える/ });
  }

  async gotoPiMessage() {
    await this.getPiMessageLink.click();
  }
}
```

`πで伝える`という1つのリンクだけを検証する現状ではこれで動く。しかしレビューで次の指摘があった。

> this.piMessageLinkではなく、引数で受け取ったほうがいいかな？
> 拡張でリストの子要素が増える

一覧に2つ目、3つ目のリンクが追加された場合、この設計だと`getPiMessageLink`と同じ形のプロパティ・メソッドをリンクの数だけ増やすことになる。

## 対応：リンク名を引数に取る

固定プロパティと固定メソッドを、リンク名を引数に取る汎用メソッドへ置き換えた。

```ts
export class PiLoopPage extends BasePage {
  readonly getList: Locator;

  constructor(page: Page) {
    super(page);
    this.getList = page.getByRole("list");
  }

  getListItemLink(name: string | RegExp): Locator {
    return this.getList.getByRole("listitem").getByRole("link", { name });
  }

  async gotoListItem(name: string | RegExp) {
    await this.getListItemLink(name).click();
  }
}
```

呼び出し側は次のようにリンク名を渡す形になる。

```ts
await expect(piLoopPage.getListItemLink(/πで伝える/)).toBeVisible();
await piLoopPage.gotoListItem(/πで伝える/);
```

一覧に新しいリンクが追加されても、`PiLoopPage`自体への変更は不要で、呼び出し側で別の名前を渡すだけで済む。

## やってみた結果

リファクタ前後で`e2e/specs/pi-loop.spec.ts`を実行し、Greenを維持した。

```text
$ npx playwright test e2e/specs/pi-loop.spec.ts --project=chromium
2 passed (4.6s)
```

呼び出し側の`e2e/specs/pi-loop.spec.ts`と`e2e/specs/vrt.spec.ts`も同じ形で更新し、型検査・Lintともにエラーなしだった。修正コミットは`d066cb3`である。

## 学んだこと

- Page Objectで「一覧の中の1項目」を表すLocatorを設計するとき、対象が将来1個から複数個に増える可能性があるなら、固定プロパティではなく識別子（名前・ID・ロールなど）を引数に取るメソッドにしておく。
- この判断はテストが1個の項目しか検証していない段階でも先取りできる。「今何個あるか」ではなく「将来何個になり得るか」で設計する。
- 汎用化のコストは低い（プロパティ2つがメソッド2つに変わるだけ）が、先送りすると項目の数だけ同型のコードが増えていく。

## 参考資料

- 根拠コミット: `d066cb3`
- 確認日: 2026-08-07

## まとめ

呼び出し側の`e2e/specs/pi-loop.spec.ts`と`e2e/specs/vrt.spec.ts`も同じ形で更新し、型検査・Lintともにエラーなしだった。修正コミットは`d066cb3`である。
