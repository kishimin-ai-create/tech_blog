# 生成画像の種類をダウンロードファイル名で検証する

## 対象読者

複数の生成モードを持つWebアプリで、PlaywrightのE2Eテストを使って「送信できた」だけでなく、要求したモードの結果が返ったことまで確認したい開発者を対象にする。

## スコープ

Mojicaの本番向けLarge E2Eで、画像タイプとPNGのダウンロードファイル名を結び付けて検証した変更を扱う。API内部の画像生成アルゴリズムや、ダウンロードできないアプリ内ブラウザーの対応は扱わない。

## 結論

拡張子だけを検証するテストでは、標準画像を要求したつもりで別の画像タイプが返る不整合を検出できない。選択した画像タイプをファイル名の契約に含め、UUID部分だけを正規表現で許容すると、モードとダウンロード結果の対応をE2Eで検証できる。

## 変更前の課題

本番E2Eは、日本語・英語それぞれで標準画像、X背景画像、Xアイコン画像を生成していた。変更前のアサートは、ファイル名が`.png`で終わることだけを確認していた。

```ts
expect(download.suggestedFilename()).toMatch(/\.png$/);
```

このアサートで確認できるのはPNG拡張子だけであり、選択した画像タイプが結果に反映されたかは確認できない。

## 実装

画像タイプをUIで使用する定義から取得し、テストの各ケースへ渡す。ファイル名は固定部分、選択タイプ、生成ごとに変わるUUID、拡張子の組み合わせとして検証する。

```ts
expect(download.suggestedFilename()).toMatch(
  new RegExp(`^mojica-${imageType}-[0-9a-f-]+\\.png$`),
);
```

ここではUUIDを固定値にしない。生成のたびに変わる値を`[0-9a-f-]+`で許容しつつ、先頭の`mojica-`、画像タイプ、末尾の`.png`は固定する。これにより、テストは実装内部のUUID生成方法ではなく、利用者に返されるダウンロード名の契約を確認する。

## 検証対象

本番Large E2Eでは、次の組み合わせを日本語・英語で実行する。

| 言語 | 画像タイプ | 期待するファイル名の形 |
| --- | --- | --- |
| 日本語・英語 | `standard` | `mojica-standard-{UUID}.png` |
| 日本語・英語 | `x-background` | `mojica-x-background-{UUID}.png` |
| 日本語・英語 | `x-icon` | `mojica-x-icon-{UUID}.png` |

各ケースでは、入力、画像タイプ選択、送信、ダウンロード名、画面キャプチャを`test.step`で分けて記録する。画像タイプの定義は`imageTypeDefinitions`から利用し、テスト側に同じ値を重複して書かない。

## 動作確認

変更対象は`frontend/e2e/tests/image-generation.large.test.ts`である。関連コミットは次のとおり。

- `8d3422c test: verify downloaded image type`

この変更により、PNG拡張子だけでなく、選択した3種類の画像タイプがファイル名へ反映されたことを確認する。実URLのE2Eでは、日本語・英語と5つのブラウザープロジェクトを組み合わせた生成・ダウンロードフローを実行した。

## 事実・判断・未確認事項

- FACT: 変更前はファイル名の`.png`拡張子だけを検証していた。
- FACT: 変更後は`mojica-{imageType}-{UUID}.png`の形を検証する。
- FACT: `imageType`は`standard`、`x-background`、`x-icon`を使用する。
- INFERENCE: 画像タイプをファイル名まで検証することで、送信操作だけでは見えない結果の取り違えを検出できる。
- ASSUMPTION: ファイル名形式が今後も公開契約として維持されることを前提にしている。形式を変更する場合は、このE2Eと利用者向け仕様を同時に更新する必要がある。

## まとめ

画像をダウンロードできたことだけでは、要求した生成モードが使われたことまでは保証できない。ファイル名に含まれる安定した意味情報を契約として検証し、UUIDのような実行ごとに変わる値だけを正規表現で許容すると、E2Eのアサートを利用者の期待に近づけられる。

## 参考

- `mojica/frontend/e2e/tests/image-generation.large.test.ts`
- `mojica/frontend/src/types/image-type.ts`
- `mojica/frontend/e2e/pages/image-generation-page.ts`
- Mojica commit `8d3422c`

確認日: 2026-09-06
