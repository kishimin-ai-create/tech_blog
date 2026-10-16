---
title: "日本語画面だけVRTが落ちたらブラウザとフォントを揃える"
tags: [Playwright, GitHubActions, VRT, Docker]
---

# 日本語画面だけVRTが落ちたらブラウザとフォントを揃える

## 結論

MojicaのNightlyで失敗した6件は、ChromeとEdgeによる日本語画面の画像比較だった。基準画像と実測画像では日本語の字体・文字幅が異なった。ブラウザとフォントを指定したコンテナを用意し、その環境で基準画像を生成し直した後、画像更新なしの再実行で45件成功した。

確定したのは描画結果が異なることと、固定環境内で比較が成功したことまでである。元の環境で最初にフォント選択を変えた上流更新は特定していない。この記事は、日本語だけが変わる視覚回帰テスト（VRT）の切り分けを扱う。

## 発生した問題

### 症状

[対象ジョブ](https://github.com/kishimin/mojica/actions/runs/38805286972/job/116664434062)は、39 passed、6 failed、30 skippedだった。6件は、ChromeとEdgeでのErrorFallback、ホーム、404の日本語画面だった。

```text
expect(page).toHaveScreenshot(expected) failed
4620 pixels ... are different.
```

ErrorFallbackは4620 pixels、ホームは7684 pixels、404は3681 pixelsの差分を記録した。各再試行でも同じ差分数だった。ダウンロードした画像を比べると、文字の描画とその周辺の幅に差があった。

### 発生条件と影響

CIは`ubuntu-latest`を使い、Playwright設定は実際のChrome・Edgeをchannelで指定していた。

```ts
{ name: "Google Chrome", use: { ...devices["Desktop Chrome"], channel: "chrome" } }
```

ロックファイルでPlaywrightの依存を固定していても、このchannelが起動するホストのChromeまで同じバージョンになるとは限らない。今回のworkflowにはブラウザパッケージや日本語フォントを明示する設定がなかった。

## 調査

### GitHub設定が原因という仮説をログで確かめる

最初はGitHub側の設定を疑う依頼だった。しかし、ジョブログでは依存サービス起動・ビルドが成功し、失敗はテストの画像比較まで進んだ後に起きていた。権限やSecrets不足を示すエラーは確認しなかった。

これはGitHub設定のあらゆる問題を否定する証拠ではない。ただし、今回の直接の失敗箇所を説明する証拠は、画像比較のログと実際の差分画像にあった。

### OSイメージの固定だけではフォント選択を説明できない

Playwright 1.62.1のnobleイメージでも確認した。最初のイメージでは次の結果だった。

```text
fc-match sans-serif:lang=ja
wqy-zenhei.ttc: "WenQuanYi Zen Hei" "Regular"
```

このまま既存の日本語基準画像へ比較すると差が出た。日本語専用フォントを追加した後も、選択順を明示しない状態では同じ候補が選ばれた。フォントが存在することと、generic familyからそのフォントが選ばれることは別だった。

## 原因

**FACT:** 基準画像とCI実測画像の日本語描画が異なった。CIはホストのブラウザを使い、フォント選択を固定していなかった。

**INFERENCE:** ブラウザ・フォントを含む描画環境の不一致が今回の失敗の有力な説明である。ただし、特定のChrome更新やフォントパッケージ更新を単独の原因として断定する比較実験は行っていない。

## 解決方法

E2E用Dockerfileで、次の要素を指定した。

| 対象 | 指定した版 |
| --- | --- |
| PlaywrightのOSイメージ | `v1.62.1-noble`、digest指定 |
| Bun | `1.3.13`、digest指定 |
| Google Chrome | `155.0.8059.39-1` |
| Microsoft Edge | `155.0.4283.45-1` |
| `fonts-noto-cjk` | `1:20230817+repack1-3` |

さらにfontconfigでgeneric sans-serifとsystem-uiの候補を指定した。次は実際の設定の一部である。

```xml
<alias>
  <family>sans-serif</family>
  <prefer><family>Noto Sans CJK JP</family></prefer>
</alias>
```

設定後の`fc-match sans-serif:lang=ja`はNoto Sans CJK JPを返した。これはその問い合わせの選択結果であり、ページ内の全要素が常に同じフォントを使うと証明するものではない。CSSで別のfamilyを指定する要素もあるため、実際の画面も確認した。

## 動作確認

日本語の基準画像15枚を生成し、英語画像が変わっていないことをGit差分で確認した。その後、更新した基準画像を含む新しいイメージをbuildし直し、snapshot更新を指定せずに実行した。

```powershell
docker build --file frontend/Dockerfile.e2e --tag mojica-e2e:verified frontend
docker run --name mojica-e2e-final --init --ipc=host --env VITE_API_URL=http://host.docker.internal:15063 mojica-e2e:verified
```

作業場所はMojicaのリポジトリルートで、Windows上のDocker Desktop Linuxコンテナと、別途起動したローカルAPIを使用した。コンテナの既定コマンドは`bun run build && bun run e2e`である。

```text
45 passed (33.2s)
30 skipped
```

30件は`RELEASE_E2E_BASE_URL`を必要とするReleaseテストである。ここで成功したのはローカルサービスを使うE2Eで、GitHub Actionsの修正後runやデプロイ先のReleaseテストまで成功したとは扱わない。

## 再発防止

push・PR・Nightlyの3ワークフローを同じE2Eイメージに揃え、基準画像を生成する手順も同じDockerfileへ寄せた。ブラウザプロジェクトのskip追加や画像比較の許容差拡大は行っていない。

今後の環境更新時には、Playwrightイメージと`bun.lock`の整合、版指定したパッケージの配信継続、基準画像の差分を確認する必要がある。バージョン指定は将来の可用性まで保証しない。

## まとめ

今回の切り分けでは、失敗したテスト名、再試行時の差分数、実際の文字描画、fontconfigの選択結果を順に確認した。日本語だけの画像差分を見つけた場合は、基準画像の更新を急ぐ前に、使うブラウザとフォント選択を再現可能にする。そのうえで、画像更新なしの比較成功を確認する。

## 参考資料

- [環境固定の実装](https://github.com/kishimin/mojica/commit/466a4bd)
- [基準画像の更新](https://github.com/kishimin/mojica/commit/abcad1b)
- [Playwright Docker公式資料](https://playwright.dev/docs/docker)
- [E2Eをコンテナへ移すときのAPI接続と成果物回収](./115-container-e2e-api-and-report-boundaries.md)

確認日: 2026年10月16日。GitHub Actionsの修正後実行は執筆時点で未確認。
