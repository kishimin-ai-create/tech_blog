# 実APIの画像生成E2EでEnter送信を検証する境界を決める

## 対象読者

Playwrightでフォーム送信をブラウザーから検証しつつ、APIモックや内部状態のアサートをE2Eへ持ち込みたくない開発者を対象にする。

## 結論

実APIを使うE2Eでは、利用者が画面で確認できる成功結果に絞り、画像生成フォームのクリック送信とEnter送信を別ケースで検証する。どちらも入力値は呼び出し元で与え、PNGダウンロードのファイル名を観測する。APIエラーの分類やフィールドエラーの詳細は、既存のRTL・MSWテストで検証する。

## なぜ実APIを使うのか

画像生成の成功ケースでは、フォーム、ブラウザー、API、バイナリレスポンス、ダウンロードという複数の境界が連続する。E2EでAPIをモックすると、HTTP接続や実際の`Content-Disposition`を検証できないため、成功経路のケースではローカル起動したAPIを使用する。

一方、決定的なエラーを実APIで再現できないケースを無理にE2Eへ置くと、環境依存の失敗を生む。APIレスポンスのコードから表示文言への変換や、422のフィールドエラー反映は、既存のコンポーネント統合テストでモックレスポンスを固定して検証する。

## クリックとEnterを分ける

Page Objectは送信手段を2つの操作として公開する。

```ts
const submit = async (): Promise<Download> => {
  const downloadPromise = page.waitForEvent("download");
  await submitButton().click();
  return downloadPromise;
};

const submitWithKeyboard = async (): Promise<Download> => {
  const downloadPromise = page.waitForEvent("download");
  await submitButton().press("Enter");
  return downloadPromise;
};
```

テストでは、両方の操作に同じ有効入力を与え、`suggestedFilename()`がPNG拡張子になることを確認する。

```ts
const download = await imageGenerationPage.submitWithKeyboard();
expect(download.suggestedFilename()).toMatch(/\.png$/);
```

これは`aria-busy`や内部のmutation状態を確認するテストではない。利用者が生成結果をダウンロードできることを、ブラウザーのダウンロードイベントという外部結果で確認している。

## テストサイズの境界

ローカルの実APIへ接続するPlaywrightテストは、DOMだけを扱うSmallではなく、ブラウザーとネットワークを含むMediumとして扱う。Small側ではフォームの入力値、アクセシブルなエラー、APIエラー表示をRTLとMSWで短時間に確認し、Medium側では実APIとの成功接続とブラウザーのダウンロードを確認する。

## 検証結果

Google Chromeプロジェクトで、通常クリックとEnter送信の画像生成E2Eを個別実行し、PNGファイル名のアサートに成功した。Prettier、`bun run typecheck`、`bun run lint`も成功した。画像生成API全ケースを一括実行した場合は、外部API応答時間に依存するタイムアウトが一度発生したため、環境依存の残存リスクとして扱う。

## 事実・判断・未確認事項

- FACT: `submit`はクリック、`submitWithKeyboard`はEnterを送信する。
- FACT: 両ケースは`Download.suggestedFilename()`のPNG拡張子を確認する。
- FACT: E2EテストはAPIレスポンスを`page.route`でモックしていない。
- INFERENCE: 成功経路を実API、決定的なエラー分類をRTLへ分けると、各層の失敗原因を切り分けやすい。
- ASSUMPTION: 実APIの可用性と応答時間は、CI環境で別途監視・調整が必要である。

## まとめ

E2Eは実ブラウザーから見える成功結果を、RTLは決定的な状態分岐を検証する。送信方法をクリックとEnterに分けても、期待する外部結果はPNGダウンロードに統一することで、テストのWhatを保てる。

## 参考

- `frontend/e2e/tests/image-generation.medium.test.ts`
- `frontend/e2e/pages/image-generation-page.ts`
- `frontend/src/features/image-generation/components/ImageGenerationForm/ImageGenerationForm.small.test.tsx`
- `C:/Users/Kazum/.codex/docs/adr/0032-separate-pure-mapping-from-transport-tests.md`
- `C:/Users/Kazum/.codex/docs/adr/0044-classify-e2e-tests-within-test-size-taxonomy.md`

確認日: 2026-09-05
