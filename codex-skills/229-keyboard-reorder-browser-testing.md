# キーボードによる並べ替えをブラウザーで確かめる

## 結論

管理画面の作品、スキル、お気に入り、リンクの並べ替えを、Tab と Enter を使うブラウザーテストで確認した。作品とリンクは別セクションの項目も用意し、対象セクションだけの ID が送信され、もう一方の表示順が保たれることを確認した。変更はドラフト PR で、アプリの main には未マージである。

## 背景

一覧のボタンをクリックするコンポーネントテストは既にあった。Issue #14 と #15 にはキーボード操作の要件があり、実際のブラウザーでも操作できることを確認する必要があった。

## 確認した操作

[PR #284](https://github.com/kishimin-ai-create/profile-hub/pull/284) では作品、[#285](https://github.com/kishimin-ai-create/profile-hub/pull/285) ではスキル、[#286](https://github.com/kishimin-ai-create/profile-hub/pull/286) ではお気に入り、[#287](https://github.com/kishimin-ai-create/profile-hub/pull/287) ではリンクを対象にした。

テストはログイン後に Tab で目的のボタンへ移動し、フォーカスを確認して Enter を押す。作品とリンクのテストでは、異なるセクションを混在させて PATCH の section と ids、および画面上の順序を確認した。

## 検証と限界

各ケースは Chromium と WebKit で成功した。API の一覧と並べ替え応答は Playwright の route で制御しているため、実 DB への保存はこのテストの対象ではない。[Nightly](https://github.com/kishimin-ai-create/profile-hub/actions/runs/42353676685) は5ジョブ成功したが、それだけで実機の支援技術や全画面のキーボード操作を証明するものではない。

## 学び

「ボタンをクリックできる」と「キーボードで到達して実行できる」は別の観測である。セクションをまたがないことも、送信内容と表示の両方で確認すると意図が明確になる。
