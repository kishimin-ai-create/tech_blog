# Blob URLと`download`属性がアプリ内ブラウザーで保証されない理由

## 対象読者

ブラウザーで生成したBlobを自動保存したいWeb開発者、特にSNSやメッセージアプリのアプリ内ブラウザーでダウンロード挙動を確認する開発者を対象にする。

## スコープ

Mojicaで実装したBlob URLと`<a download>`によるPNG保存、通常のPlaywrightブラウザーでの確認、LINE内ブラウザーで自動保存できなかった観測結果を扱う。LINEの内部実装や、すべてのアプリ内ブラウザーに共通する解決策は断定しない。

## 結論

Blob URLと`<a download>`は、標準的なブラウザーAPIを使った妥当な実装である。しかし、MDNが説明するように`download`の扱いはブラウザー、ユーザー設定、保存方法によって変わるため、アプリ内ブラウザーで自動保存されることまでは保証されない。Mojicaでは通常のPlaywright対象ブラウザーでPNGダウンロードを確認できた一方、LINE内ブラウザーでは自動保存に失敗した。この差は、アプリ内ブラウザーを通常ブラウザーと同じダウンロード境界として扱えない制約を示している。

## 実装の境界

画像生成APIはPNGのバイナリを返す。フロントエンドではレスポンスをBlobとして受け取り、オブジェクトURLを作成して一時的なアンカーをクリックする。

```ts
const objectUrl = URL.createObjectURL(blob);
const anchor = document.createElement("a");
anchor.href = objectUrl;
anchor.download = fileName;
document.body.append(anchor);
anchor.click();
anchor.remove();
URL.revokeObjectURL(objectUrl);
```

`document.body`へ追加してからクリックするのは、ブラウザーのDOM上に存在するアンカーとして操作するためである。保存処理が終わった後に要素とオブジェクトURLを破棄し、一時リソースを残さない。

## MDNから確認できること

MDNの`HTMLAnchorElement.download`の説明では、`download`はリンク先を表示ではなくダウンロードとして扱う意図を示す属性で、指定値は推奨ファイル名になる。ただし、指定値が実際に使われることや、ダウンロードが必ず起きることを判定するための属性ではないと説明されている。

また、`<a>`要素の`download`の説明では、同一オリジン、`blob:`、`data:`スキームで利用できる一方、ブラウザー、ユーザー設定、その他の要因によって、保存・表示・外部アプリ起動などの扱いが変わるとされている。Blob URL自体は`URL.createObjectURL()`で作成し、不要になったら`URL.revokeObjectURL()`で解放するのが基本である。

これらは「APIの使い方が間違っていなければ、どのホスト環境でも自動保存できる」という意味ではない。`download`は保存を要求するための宣言であり、最終的なUIや保存動作はブラウザー実装の境界にある。

## LINE内ブラウザーでの観測

Mojicaでは、通常のPlaywright対象ブラウザーでPNGのダウンロードを確認した。しかし、LINE内ブラウザーで同じBlob URLと`<a download>`による自動保存を実行したところ、期待した自動保存にならなかった。

この結果から確定できるのは、次の範囲である。

- MojicaのBlob生成とアンカークリックは、通常の対象ブラウザーでは動作した。
- LINE内ブラウザーでは、同じ方法で自動保存できなかった。
- MDNの説明どおり、ダウンロード結果はブラウザー環境に依存する。

LINE内ブラウザーが内部でどのAPIを制限しているか、また端末やLINEのバージョンごとに挙動が異なるかは、この確認だけでは確定できない。したがって、特定の内部原因を断定してはいけない。

## 対応方針

Webアプリ側では、標準APIに沿った保存処理を実装し、通常ブラウザーと対象端末での結果を分けて確認する。アプリ内ブラウザーで自動保存できない場合は、次のような代替導線を検討する。

- 画像を新しい画面で表示し、ユーザーに長押し保存を案内する
- 外部ブラウザーで開く導線を提供する
- 生成画像をサーバー上の同一オリジンURLから取得する設計を検討する

これらは追加のUX・セキュリティ・保存期限の設計を必要とするため、今回の自動保存実装の修正だけで解決したとは扱わない。

## 事実・判断・未確認事項

- FACT: MojicaはAPIのPNGレスポンスをBlob URLに変換し、`<a download>`をクリックして保存する。
- FACT: `URL.createObjectURL()`後にアンカーを`document.body`へ追加してクリックし、要素とURLを破棄する。
- FACT: 通常のPlaywright対象ブラウザーではPNGダウンロードに成功した。
- FACT: LINE内ブラウザーでは同じ方法の自動保存に失敗した。
- FACT: MDNは`download`の結果がブラウザー、ユーザー設定、その他の要因で変わると説明している。
- INFERENCE: アプリ内ブラウザーの自動保存は、標準APIの実装だけでは保証できない。
- ASSUMPTION: LINE内ブラウザーでの失敗原因が、Blob URL、クリック起点、保存UI、端末設定のどれかは未確認である。

## まとめ

Blob URLと`<a download>`は、Webブラウザーでバイナリを保存する標準的な境界である。ただし、`download`は保存結果を保証するAPIではない。通常ブラウザーでの成功とLINE内ブラウザーでの失敗を分けて記録し、必要なら外部ブラウザーや手動保存の導線を別の要件として設計する必要がある。

## 参考

- [MDN: HTMLAnchorElement.download](https://developer.mozilla.org/en-US/docs/Web/API/HTMLAnchorElement/download)
- [MDN: `<a>`要素の`download`属性](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/a#download)
- [MDN: URL.createObjectURL()](https://developer.mozilla.org/en-US/docs/Web/API/URL/createObjectURL_static)
- [MDN: blob: URLs](https://developer.mozilla.org/en-US/docs/Web/URI/Reference/Schemes/blob)
- `mojica/frontend/src/features/image-generation/utils/downloadGeneratedImage.ts`
- `mojica/frontend/e2e/tests/image-generation.large.test.ts`

確認日: 2026-09-06
