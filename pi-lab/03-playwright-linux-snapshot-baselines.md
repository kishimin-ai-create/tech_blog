# PlaywrightでLinux用の基準画像がないエラーを解決する

## 結論

基準画像生成コマンドが成功しても、それだけでは比較が安定している証明にならない。同じ環境で更新フラグを外して再実行する。

## 発生した問題

WindowsでVRTの基準画像を作ったあと、UbuntuのGitHub Actionsで次のエラーが発生した。

```text
Error: A snapshot doesn't exist at
e2e/specs/vrt.spec.ts-snapshots/05-pi-message-retry-chromium-linux.png,
writing actual.
```

これはテストコードの探索失敗ではない。PlaywrightがLinux用の期待画像を探したものの、リポジトリには`*-chromium-win32.png`しか存在しなかった状態である。

## なぜOS名が付くのか

Playwrightの画像スナップショット名には、プロジェクト名とプラットフォームが含まれる。ブラウザの文字描画やアンチエイリアスはOSやフォントで変わるため、Windows用画像をLinux用としてコピーするのは適切ではない。

pi-labではPlaywright 1.61.1のUbuntu Noble公式イメージでLinux画像を生成した。

## ホストのnode_modulesをコンテナへ持ち込まない

Windowsの`node_modules`をLinuxコンテナから直接使うと、OS固有バイナリの不整合が起きる可能性がある。そこで、リポジトリをbind mountしつつ、`node_modules`だけDocker volumeで隠した。

PowerShellからの実行例は次のとおり。

```powershell
$repo = (Get-Location).Path
$volume = "pi-lab-pw-1611-node-modules"

docker volume create $volume
docker run --rm --init --ipc=host `
  -v "${repo}:/work" `
  -v "${volume}:/work/node_modules" `
  -w /work `
  mcr.microsoft.com/playwright:v1.61.1-noble `
  bash -lc "npm ci && npx playwright test e2e/specs/vrt.spec.ts --update-snapshots"
```

この操作はリポジトリ内へLinux用PNGを書き込む。一方、依存関係は専用volumeへ入るため、Windows側の`node_modules`を置き換えない。

## 生成後は更新なしで再比較する

基準画像生成コマンドが成功しても、それだけでは比較が安定している証明にならない。同じ環境で更新フラグを外して再実行する。

```powershell
docker run --rm --init --ipc=host `
  -v "${repo}:/work" `
  -v "${volume}:/work/node_modules" `
  -w /work `
  mcr.microsoft.com/playwright:v1.61.1-noble `
  bash -lc "npx playwright test e2e/specs/vrt.spec.ts"
```

pi-labでは最初にChromiumのLinux基準画像5枚を追加し、更新なし比較で`1 passed`を確認した。コミットは`980aa24`である。

## 失敗しやすい対応

### Windows画像をLinux名へコピーする

ファイル不足は解消しても、描画差で失敗する。基準画像は実際の比較環境で生成する。

### CIで毎回`--update-snapshots`を使う

差分を失敗として検出できなくなる。更新はレビュー作業、通常実行は検証作業として分離する。

### `maxDiffPixels`を先に増やす

環境差の原因を隠す可能性がある。まずExpected、Actual、Diffを確認し、同一環境化を検討する。

## 参考資料

- [Playwright: Visual comparisons](https://playwright.dev/docs/test-snapshots)
- [Playwright: Docker](https://playwright.dev/docs/docker)
- 根拠コミット: `980aa24`
- 確認日: 2026-08-06

## まとめ

環境差の原因を隠す可能性がある。まずExpected、Actual、Diffを確認し、同一環境化を検討する。
