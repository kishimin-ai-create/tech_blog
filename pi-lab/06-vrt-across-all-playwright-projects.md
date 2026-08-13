# VRTを全Playwrightプロジェクトへ拡張する

## はじめに

したがって最終的な基準画像は50枚になる。

## 結論

Playwright設定に複数のブラウザや端末プロジェクトがあるのに、VRTだけをChromiumへ限定すると、機能テストと視覚テストの保証範囲がずれる。pi-labでは条件付き`test.skip()`を削除し、設定済みの全5プロジェクトで同じ主要導線を画像比較する構成へ変更した。

## 変更前に発生していた4件のskip

VRTには次の条件があった。

```ts
test.skip(
  testInfo.project.name !== "chromium",
  "VRTの基準画像はデスクトップChromiumに固定する",
);
```

Playwright設定には5プロジェクトがあるため、Chromium以外の4件が設計どおりskipされていた。

| プロジェクト | VRT変更前 | VRT変更後 |
| --- | --- | --- |
| chromium | 実行 | 実行 |
| webkit | skip | 実行 |
| Mobile Chrome | skip | 実行 |
| Mobile Safari | skip | 実行 |
| Microsoft Edge | skip | 実行 |

## skipを消すだけでは完成しない

条件を削除した最初の実行では、WebKit、Mobile Chrome、Mobile Safariについて、各5枚の基準画像不足が発生した。これは期待したRedである。

```text
3 failed
Error: A snapshot doesn't exist at ...-webkit-linux.png, writing actual.
```

その後、固定Linuxコンテナで全プロジェクトの基準画像を生成し、更新なしで再比較した。Windowsローカル実行用の基準画像も同じ手順で生成した。

## 基準画像数を見積もる

今回の軸は次の3つである。

- 表示状態: 5
- Playwrightプロジェクト: 5
- 管理対象OS: Windows、Linuxの2つ

したがって最終的な基準画像は50枚になる。

```text
5 states × 5 projects × 2 platforms = 50 images
```

既にChromium用が10枚あったため、追加した画像は40枚だった。プロジェクトを増やすたびに、表示状態数×OS数だけ基準画像とレビュー量が増える。

## モバイル画像を別契約として扱う

Mobile Chromeは393×727、Mobile Safariは390×664のfull-page画像になった。デスクトップと同じDOMを使っていても、処理中メッセージの折り返しやヘッダーの寸法は異なる。

この差は許容誤差ではなく、端末ビューポートごとの期待表示である。そのため、デスクトップ画像をリサイズして流用せず、各プロジェクトで基準画像を生成した。

## やってみた結果

固定Ubuntu Nobleコンテナで次を確認した。

```text
VRT: 5 passed
All E2E: 20 passed
Skipped: 0
```

WindowsでもVRT 5件が更新なしで成功した。新規画像は目視し、一覧、入力、処理中、結果、再試行の状態に表示崩れがないことを確認した。変更コミットは`10f920a`である。

## 設計上の判断

検出範囲を広げる代わりに、保存容量、実行時間、基準画像更新時のレビュー量は増える。今後プロジェクトを追加する場合は、機能E2EだけでなくVRT用の基準画像も必要になる。

この判断はADR-0015として記録し、Chromium限定だったADR-0013を置き換えた。

## 参考資料

- [Playwright: Visual comparisons](https://playwright.dev/docs/test-snapshots)
- 根拠コミット: `10f920a`
- 確認日: 2026-08-06

## まとめ

この判断はADR-0015として記録し、Chromium限定だったADR-0013を置き換えた。
