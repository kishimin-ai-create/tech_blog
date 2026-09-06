# UI変更後にLinux用Visual Regressionベースラインを更新する

## 対象読者

Playwrightを複数ブラウザープロジェクトで実行し、CI上のVisual Regression Test（VRT）を運用する開発者を対象にする。

## スコープ

Mojicaでカラーピッカーの境界線を変更した後、CIのVRTが失敗した原因を切り分け、Linux用ベースラインを更新した手順を扱う。VRTの導入方法やデザイン差分の自動判定そのものは扱わない。

## 結論

今回のVRT失敗は、実装変更に対してLinux用の期待画像10枚が古いままだったことが原因だった。画面差分を確認したうえで、日本語・英語と5つのPlaywrightプロジェクトに対応するLinuxベースラインだけを更新すると、CIのVRTを成功させられた。

## 発生した問題

### 症状

Pull RequestのCIで、画像生成ホームのVRTが10件失敗した。失敗対象は日本語・英語のホーム画面で、NotFound画面のVRTは失敗していなかった。

### 発生条件

- UI変更: カラーピッカー入力の境界線トークンを`border-border`から`border-input`へ変更
- テスト: `frontend/e2e/tests/visual-regression.medium.test.ts`
- 対象: Google Chrome、Microsoft Edge、Safari、Android Chrome、iPhone Safari
- ベースライン: `frontend/e2e/tests/visual-regression.medium.test.ts-snapshots/`

### 影響

実装変更自体が意図した表示差分であっても、期待画像が更新されていないためCIが失敗した。VRTの失敗を実装不具合と判断して変更を戻すと、レビュー済みのUI変更まで失われる可能性がある。

## 調査

まず、失敗したテスト名と差分画像を確認し、すべての失敗が同じ画面領域に集中しているかを調べた。失敗は画像生成ホームのスクリーンショットに限られ、差分はカラーピッカー周辺の境界線表示だった。

次に、直前の実装コミットを確認した。

```text
git show --stat 1cf199b
git show --stat ea146c6
```

`1cf199b`ではカラーピッカーを含む共有入力の境界線表示が変更されていた。したがって、CIが撮影した新しい表示が誤っているのではなく、Linux用の期待画像が変更前のままだったと判断した。

## 原因

VRTは現在の画面を既存のPNGベースラインと比較する。UIの境界線トークンを変更すると、機能やアクセシビリティの振る舞いが変わらなくても画像のピクセルが変化する。今回はその変更に対応するLinuxベースラインの更新が同じ変更として行われていなかった。

## 解決方法

日本語・英語それぞれについて、5つのPlaywrightプロジェクトのLinuxベースラインを更新した。変更したのは画像生成ホームの10枚で、NotFound画面やWindows用ベースラインは対象に含めなかった。

```text
frontend/e2e/tests/visual-regression.medium.test.ts-snapshots/
  image-generation-home-*-linux.png
  image-generation-home-en-*-linux.png
```

ベースライン更新は、差分が意図したUI変更と一致することを確認した後に行う。差分画像を確認せずに全ベースラインを一括更新すると、予期しない回帰を期待画像へ固定する危険がある。

## 動作確認

修正後、ローカルVRTを実行し、対象の比較が成功することを確認した。さらにCIを再実行し、VRTを含むFrontend Mediumジョブと、Small・API・Storybook・Coverageのジョブが成功した。

関連コミット:

- `1cf199b fix: align color picker field styling`
- `ea146c6 test: update Linux visual baselines`

## 再発防止

- UI変更と、影響する実行環境のベースライン更新を同じ変更単位でレビューする。
- ベースライン更新前に、差分画像が仕様変更の範囲に収まっていることを確認する。
- ベースライン更新後は、単一プロジェクトだけでなくCIと同じ全プロジェクトでVRTを実行する。
- WindowsとLinuxの画像を混同せず、実行環境ごとのベースラインを個別に管理する。

## まとめ

- 症状: 画像生成ホームのVRTが10件失敗した。
- 原因: カラーピッカーの表示変更に対してLinux用ベースラインが古かった。
- 解決: 意図した差分を確認し、対象となるLinux画像10枚だけを更新した。
- 教訓: VRTのベースライン更新は、差分をレビューしたうえで実行環境単位に限定する。

## 参考

- `mojica/frontend/e2e/tests/visual-regression.medium.test.ts`
- `mojica/frontend/e2e/tests/visual-regression.medium.test.ts-snapshots/`
- `mojica/frontend/src/components/ui/input.tsx`
- Mojica commit `1cf199b`
- Mojica commit `ea146c6`

確認日: 2026-09-06
