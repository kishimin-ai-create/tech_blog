# ReactフォームのPNG自動ダウンロードをブラウザAPI境界でテストする

## 対象読者

APIからBlobを受け取り、ブラウザのダウンロードを開始するReact機能をテストする開発者。

## スコープ

Mojicaの画像生成フォームで、成功レスポンスからダウンロードが開始されることをSmallテストで確認する方法を扱う。実ブラウザでのファイル保存先検証やE2Eテストは扱わない。

## 課題

`POST /images`の成功結果はPNGのバイナリであり、画面にプレビューを表示するのではなく自動ダウンロードする仕様だった。テスト環境では実際のファイル保存を前提にできないため、アプリが利用するブラウザAPIとの境界を検証対象にした。

## テスト方法

成功レスポンスを生成済みMSWハンドラーで返し、`URL.createObjectURL`、`URL.revokeObjectURL`、アンカー要素の`click`をスパイする。生成されたアンカーの`href`と`download`属性を確認することで、サーバーのファイル名を使ったダウンロード開始を表現できる。

```ts
vi.spyOn(URL, "createObjectURL").mockReturnValue(objectUrl);
vi.spyOn(URL, "revokeObjectURL").mockImplementation(() => undefined);
vi.spyOn(HTMLAnchorElement.prototype, "click").mockImplementation(
  () => undefined,
);

worker.use(
  getPostImagesMockHandler(() => new Uint8Array([137, 80]).buffer),
);
```

テストではフォームへ有効な値を入力して送信し、クリックが一度行われたこと、ダウンロード名が`generated-image.png`であること、Object URLがアンカーへ設定されたことを確認する。これは`downloadGeneratedImage`の内部実装そのものではなく、利用者が成功後にファイル取得を開始できる契約を検証する境界である。

## なぜブラウザAPIを差し替えるのか

JSDOMでは、実ブラウザのダウンロード処理やファイルシステムへの保存をSmallテストの前提にできない。一方で、Object URLの生成とアンカークリックはアプリが依存する外部境界なので、そこをテストダブルで観測すれば、ネットワークからダウンロード開始までの処理を決定的に確認できる。

## 事実・判断・未確認事項

- FACT: `ImageGenerationForm.small.test.tsx`にPNG成功レスポンスの自動ダウンロードテストを追加した。
- FACT: テストは生成済みMSWハンドラー、Object URL、アンカーの`click`を利用している。
- INFERENCE: Smallテストではファイルシステムを検証せず、ブラウザAPI境界とダウンロード属性を検証する分割が適切である。
- 未確認事項: 実ブラウザで実際のファイルが保存されること、Content-Dispositionの複数形式は今回確認していない。

## まとめ

Blobダウンロードは、ネットワーク応答とブラウザAPIの境界を分けてテストすると安定する。Smallテストでは「ダウンロードを開始した」という利用者向け契約を、クリック回数とダウンロード属性で確認し、ファイルシステム確認は別のテスト層へ委ねる。

## 参考

- `frontend/src/features/image-generation/components/ImageGenerationForm/ImageGenerationForm.small.test.tsx`
- `frontend/src/features/image-generation/utils/downloadGeneratedImage.ts`
- `frontend/src/api/endpoints/image/image.msw.ts`
- `5cedcee test: verify automatic PNG downloads`
