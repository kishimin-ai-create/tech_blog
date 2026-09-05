# Playwrightのfixtureと関数形式Page ObjectでE2Eの責務を分離する

## 対象読者

Playwright Testで、テスト本体にロケーターやブラウザー初期化処理を散在させず、画面操作を再利用したい開発者を対象にする。

## 結論

fixtureはテストへ提供するライフサイクルと共通設定を担当し、Page Objectは利用者が行う画面操作を関数として公開する。Mojicaでは、localeの初期化とPage Objectの生成を`frontend/e2e/fixtures/test.ts`に集約し、画面ごとの操作は`frontend/e2e/pages`へ分離した。

## fixtureに置く責務

Playwrightのfixtureは、テストごとに必要な`Page`へロケールを初期化し、対応するPage Objectを提供する。テストは`test`を専用エントリーポイントからimportするだけでよく、`localStorage`の初期化を各ケースで繰り返さない。

```ts
export const test = base.extend<E2eOptions & E2eFixtures>({
  locale: async ({ browserName: _browserName }, provide) => {
    void _browserName;
    await provide("ja");
  },
  imageGenerationPage: async ({ page, locale }, provide) => {
    await page.addInitScript((selectedLocale) => {
      localStorage.setItem("locale", selectedLocale);
    }, locale);
    await provide(imageGenerationPage(page, locale));
  },
});
```

ここで`provide`はPlaywrightがfixture値をテストへ渡すための公式のコールバックである。`locale`はfixtureの既定値として提供し、シナリオ固有の値をPage Objectへ埋め込まない。

## Page Objectに置く責務

Page Objectは、ロケーターと利用者の操作を関数としてまとめる。例えば画像生成画面は、入力値を呼び出し元から受け取り、送信操作の結果として`Download`を返す。

```ts
const fillText = async (value: string) => {
  await textInput().fill(value);
};

const submit = async (): Promise<Download> => {
  const downloadPromise = page.waitForEvent("download");
  await submitButton().click();
  return downloadPromise;
};
```

`fillText("KA")`のようなシナリオ値はテスト側に残す。Page Objectが値を固定すると、別の入力境界を検証したいテストから再利用できないためである。

## ロケール対応セレクター

画面内の表示名はロケールごとの対応表にまとめ、Page Objectが受け取ったlocaleから選択する。テストは日本語・英語の文言定義を直接参照せず、操作の意図だけを呼び出す。

```ts
type LocalizedSelector = Record<Locale, RegExp>;

export const imageGenerationSelectors = {
  heading: {
    ja: /文字で、文字を描く。/,
    en: /Draw with text, draw text\./,
  },
} satisfies Record<"heading", LocalizedSelector>;
```

Playwrightの`getByRole`は名前に正規表現を受け取れるため、句読点や翻訳の差分をPage Object内で吸収できる。選択肢の追加時は対応表へlocale値を追加し、テストの操作コードは変更しない。

## 検証結果

対象ブランチでは、fixtureを`frontend/e2e/fixtures/test.ts`へ整理し、画像生成・404・エラーフォールバックのPage Objectをfixtureから提供した。`bun run typecheck`、`bun run lint`、Prettierチェックは成功した。実APIを使う画像生成の通常クリックとEnter送信のE2Eは、Google Chromeプロジェクトで個別に成功した。

## 事実・判断・未確認事項

- FACT: fixtureはlocale初期化とPage Object提供を担当する。
- FACT: Page Objectの入力関数は値を引数で受け取る。
- FACT: E2Eテストは`../fixtures/test.ts`から`test`と`expect`をimportする。
- INFERENCE: この分離により、ロケーター変更とシナリオ変更の影響範囲を分けやすい。
- ASSUMPTION: fixtureのlocale既定値を将来テスト設定から上書きする場合は、別のテスト設定が必要になる。

## まとめ

共通の初期化はfixture、画面の利用者操作は関数形式Page Object、具体的な入力値と期待結果はテスト本体に置く。これにより、E2Eコードは「何を検証するか」を保ち、画面構造の変更はPage Objectへ閉じ込められる。

## 参考

- `frontend/e2e/fixtures/test.ts`
- `frontend/e2e/pages/image-generation-page.ts`
- `frontend/e2e/tests/image-generation.medium.test.ts`
- `C:/Users/Kazum/.codex/docs/adr/0014-provide-page-objects-as-playwright-fixtures.md`
- `C:/Users/Kazum/.codex/docs/adr/0053-pass-scenario-values-to-e2e-page-objects.md`

確認日: 2026-09-05
