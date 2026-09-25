# GitHub ActionsでVRTが突然失敗したら、ubuntu-latestのブラウザー更新を確認する

## 要約

アプリケーションのコミットが変わっていないのにPlaywrightのVisual Regression Test（VRT）が失敗した場合、期待画像だけでなくGitHub-hosted runnerのイメージとブラウザーのバージョンも比較する。Mojicaでは成功実行と失敗実行のアプリケーションSHAが同じままUbuntuイメージが更新され、ChromeとEdgeがメジャーアップデートされた時点でLinuxのVRTが失敗し始めた。

## 発生した問題

2026年9月22日のNightlyは成功し、9月23日から日本語のホーム、NotFound、ErrorFallback画面でChromeとEdgeのVRTが失敗した。最初に失敗した実行と、その前の成功実行は同じMojica commitをcheckoutしていた。

Nightlyログで注目するエラーは次のとおりだった。

```text
Error: expect(page).toHaveScreenshot(expected) failed
3681 pixels (ratio 0.01 of all image pixels) are different.
```

このメッセージは、Playwrightの画面取得やアサーション実行そのものがクラッシュしたのではなく、現在の画面と保存済みスナップショットで画素が異なったことを示す。ログにはExpected画像、Received画像、Diff画像へのパスも表示されるため、差分画像を見て意図しないUI回帰か、環境由来の描画差かを調べる。

## 調査

まず成功実行と最初の失敗実行のcommit SHAを比較した。どちらも`8d24b83251158c037def59b2dc642c527ef54d3b`で、アプリケーションコードは同一だった。

次に、ログの`Runner Image`を比較した。

| 実行 | 結果 | Ubuntu image |
| --- | --- | --- |
| 2026-09-22 | 成功 | `20260907.300.1` |
| 2026-09-23 | 失敗 | `20260920.314.1` |

GitHubのrunner imageリリースノートでは、更新によってGoogle Chromeが`152.0.7977.82`から`153.0.8010.52`へ、Microsoft Edgeが`152.0.4191.66`から`153.0.4234.48`へ変わったと確認できる。`ubuntu-latest`はOSのラベルであり、同じworkflow設定でもホストイメージとそこに含まれるブラウザーのバージョンが時間とともに更新される。

## 原因の判断

### FACT

- 直前の成功実行と最初の失敗実行でMojicaのcommit SHAは同じだった。
- runner imageは`20260907.300.1`から`20260920.314.1`へ変わっていた。
- 新イメージではChromeとEdgeが152系から153系へ更新されていた。
- 差分は日本語画面のChrome/Edge用Linuxスナップショットで再現し、再試行時も安定していた。

### INFERENCE

VRT失敗の発生時点とブラウザー更新が一致しているため、新しいブラウザーまたはrunner image環境での描画変化が主因である可能性が高い。ただし、OSパッケージなどrunner image内の全変更を一つずつ切り離していないため、「Chrome/Edgeの更新だけが原因」とまでは確定していない。

## 対応

差分画像とテスト結果を確認し、失敗実行で安定して撮影された実画像を使って、該当するLinuxベースライン6枚を更新した。

```text
image-generation-home-Google-Chrome-linux.png
image-generation-home-Microsoft-Edge-linux.png
not-found-Google-Chrome-linux.png
not-found-Microsoft-Edge-linux.png
error-fallback-Google-Chrome-linux.png
error-fallback-Microsoft-Edge-linux.png
```

画像更新では、意図しないUIの変化をそのまま期待値に固定しないよう、Expected/Received/Diffを確認する。OSやブラウザーごとのbaselineを混ぜない。

## 別のエラーを混同しない

同じNightlyでは、iPhone Safariの画像生成テストも失敗したが、これはVRTとは別の症状だった。

```text
Test timeout of 30000ms exceeded.
waiting for event "download"
```

APIログでは旧Glyph Forge revisionへの呼び出しがHTTP 429になっていた。ブラウザーがダウンロードイベントを受け取れなかった結果として待機がタイムアウトしたため、スナップショット更新だけでは直らない。Glyph Forgeのpinも共有IPレート制限を取り除いたrevisionへ更新し、ローカルのiPhone Safari対象テスト2件が通ることを確認した。

## 検証と制約

- 成功実行と失敗実行でアプリケーションSHAが同じであることを確認した。
- runner imageのリリースノートでブラウザーのバージョン差を確認した。
- 差分画像を更新し、ローカルの対象iPhone Safari E2Eを2件実行して成功した。
- 修正後のGitHub Actions全体はまだ実行されていない。
- Windowsのローカル全E2Eでは別のプラットフォーム画像比較が6件失敗しており、今回のLinux baseline更新の検証とは区別している。

## 学び

CIの視覚テストが突然赤くなったら、まずアプリの差分、テストの差分、ホスト環境の差分を同じ時系列で確認する。`expect(...).toHaveScreenshot`のpixel差分と、`waitForEvent("download")`のタイムアウトは原因層が異なるため、テスト名とログの直前のAPI応答を併せて見る。`ubuntu-latest`を使う限りホスト環境は固定されないので、runner imageのバージョンとブラウザーの更新記録を障害時に比較できるようにしておく。

## 参考資料

- [直前の成功Nightly（2026-09-22）](https://github.com/kishimin/mojica/actions/runs/35782532497)
- [最初に失敗したNightly（2026-09-23）](https://github.com/kishimin/mojica/actions/runs/35919503318)
- [Nightly失敗ログ（2026-09-24）](https://github.com/kishimin/mojica/actions/runs/36058980884)
- [Ubuntu 24.04 runner image 20260920.314の更新内容](https://github.com/actions/runner-images/releases/tag/ubuntu24%2F20260920.314)

確認日: 2026-09-26
