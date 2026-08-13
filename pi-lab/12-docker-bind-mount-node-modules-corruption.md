# VRT基準画像の更新で、Dockerのbind mountがWindowsのnode_modulesを壊した話

## はじめに

`playwright-report`のDiff画像を確認すると、差分はロゴ画像の領域だけに集中していた（解析手順は[Playwright Artifactから画像差分の原因を調べる](./04-playwright-artifact-image-diff-analysis.md)と同じ）。モバイル系プロジェクトの方が差分比率が大きいのは、ビューポートに対するロゴの相対サイズが大きいためである。

## 結論

Docker公式Playwrightイメージで基準画像を再生成するとき、ホストの実プロジェクトディレクトリをそのまま`-v`でbind mountして`npm ci`を実行すると、コンテナ内のLinux版npmがホスト側の`node_modules`をLinux向けに書き換えてしまう。Windows側の`npx`が直後に壊れる。

## 発端：意図した資産変更が基準画像に反映されていなかった

前記事の`db2dabb`はヘッダーロゴをPNG（2.1MB）からSVG（268KB）へ切り替えた。しかしVRT（Visual Regression Testing）の基準画像は、このロゴ切り替えより前のコミット（`3bf6d6c`、`980aa24`）で生成されたままだった。

CI（GitHub Actions、Ubuntu Nobleコンテナ）で`npx playwright test`を実行すると、VRT対象の4プロジェクトすべてで`01-pi-loop.png`が失敗した。

```text
[chromium]       3309 pixels (ratio 0.01) are different.
[webkit]         3172 pixels (ratio 0.01) are different.
[Mobile Chrome]  13326 pixels (ratio 0.05) are different.
[Mobile Safari]  13462 pixels (ratio 0.06) are different.
```

`playwright-report`のDiff画像を確認すると、差分はロゴ画像の領域だけに集中していた（解析手順は[Playwright Artifactから画像差分の原因を調べる](./04-playwright-artifact-image-diff-analysis.md)と同じ）。モバイル系プロジェクトの方が差分比率が大きいのは、ビューポートに対するロゴの相対サイズが大きいためである。

これはADR-0013が定める「意図したデザイン変更では、差分を確認したうえで基準画像を更新する」に該当するケースであり、`db2dabb`でロゴ形式を切り替えた際に基準画像の更新が漏れていたことが原因だった。

## workflow_dispatchでの自動更新は404で失敗した

CIと同じコンテナで基準画像を更新するため、`workflow_dispatch`で手動起動できるGitHub Actionsジョブを新規に用意した（`.github/workflows/update-vrt-snapshots.yml`）。

```yaml
on:
  workflow_dispatch:
jobs:
  update-snapshots:
    container:
      image: mcr.microsoft.com/playwright:v1.61.1-noble
    steps:
      - run: npx playwright test e2e/specs/vrt.spec.ts --update-snapshots
      # ...更新結果をコミット
```

しかし`gh workflow run update-vrt-snapshots.yml --ref pi-message`は次のエラーで失敗した。

```text
HTTP 404: workflow update-vrt-snapshots.yml not found on the default branch
```

GitHubの仕様上、`workflow_dispatch`はワークフローファイルがデフォルトブランチ（`main`）に存在しないとAPI/CLIから起動できない。作業ブランチにしかファイルがない状態では、マージするまでこの経路は使えない。

## ローカルDockerでの更新中にnode_modulesが壊れた

そこでREADMEに既存のDocker手順を使った。

```bash
docker run --rm --network host -v "$(pwd):/work" -w /work \
  mcr.microsoft.com/playwright:v1.61.1-noble \
  bash -c "npm ci && npx playwright test e2e/specs/vrt.spec.ts --update-snapshots"
```

これ自体は成功し、Linux用の基準画像16枚（一覧・入力・結果・再試行 × 4プロジェクト。ヘッダーを持たない処理中画面は対象外）が再生成された。

問題はこの直後に起きた。同じWindows環境で続けて次を実行すると、

```text
$ npx playwright --version
'playwright' is not recognized as an internal or external command,
operable program or batch file.
```

`npx`が壊れていた。

### 原因

`-v "$(pwd):/work"`は、ホストの実プロジェクトディレクトリをそのままコンテナへbind mountする。コンテナ内で実行した`npm ci`は、Dockerの仮想ファイルシステムではなく**ホスト側の本物の`node_modules`**へ直接書き込む。

npmが`node_modules/.bin/`に作る実行用シムは、OSによって中身が異なる。

| OS | 生成されるファイル |
| --- | --- |
| Windows | `playwright`（拡張子なし）に加え、`playwright.cmd`・`playwright.ps1` |
| Linux | POSIXシンボリックリンク`playwright -> ../@playwright/test/cli.js`のみ |

コンテナ内のLinux版npmが`node_modules`を作り直した結果、Windows用の`.cmd`/`.ps1`が消え、Unix向けシンボリックリンクだけが残った。Windows側の`npx`は`.cmd`のような拡張子付き実行ファイルを解決しようとするため、見つからずエラーになった。

```bash
$ ls -la node_modules/.bin/playwright*
lrwxrwxrwx 1 Kazum 197609 26 ... node_modules/.bin/playwright -> ../@playwright/test/cli.js
```
（`.cmd`が存在しないことを示す）

### 復旧

Windows側で`node_modules`を作り直した。

```bash
rm -rf node_modules && npm ci
npx playwright --version
# Version 1.61.1
```

## 基準画像の更新を完了する

node_modulesの復旧後、残っていたWindows用の基準画像もローカルで更新した。

```bash
npx playwright test e2e/specs/vrt.spec.ts --update-snapshots
```

Linux用16枚（Docker）、Windows用16枚（ローカル）、合計32枚を更新した。更新後、目視で2枚をサンプル確認しSVGロゴが正しく描画されていることを確かめたうえで、全体検証を行った。

```text
$ npx playwright test
  19 passed (8.4s)

$ npx vitest run
  Test Files  12 passed (12)
      Tests  26 passed (26)

$ npx tsc --noEmit
（エラーなし）
```

修正コミットは`5700d01`である。

## 学んだこと

- ホストの実ディレクトリをbind mountしたコンテナ内で`npm install`/`npm ci`を実行すると、ホストとコンテナのOSが異なる場合に`node_modules`の実行用シムが壊れる。**基準画像の再生成に必要なのはコンテナ内のPlaywright実行環境であって、`node_modules`のインストール先までホストと共有する必要はない**。
- 再発防止には、`docker run`時に`node_modules`だけ別の匿名ボリュームへ逃がす（`-v /work/node_modules`を追加してbind mount経路から除外する）か、リポジトリを一時ディレクトリへコピーしてからコンテナに渡す方法が考えられる。今回はまだ未適用であり、次にDockerでVRTを更新する際に検証が必要である。
- `workflow_dispatch`はデフォルトブランチにマージされるまでAPI/CLIから起動できないため、作業ブランチだけで完結する検証手段として当てにできない。ローカルDocker、またはpush/PRトリガーのワークフローを代替手段として持っておく。

## 未確認事項

- `-v`に匿名ボリュームを追加してbind mount経路からnode_modulesを除外する対策は、まだ実装・検証していない。
- `update-vrt-snapshots.yml`の`workflow_dispatch`は、`main`へマージされた後の動作を未確認である。

## 参考資料

- [Playwright: Docker](https://playwright.dev/docs/docker)
- [GitHub Docs: Manually running a workflow](https://docs.github.com/en/actions/how-tos/manage-workflow-runs/manually-run-a-workflow)
- [npm docs: bin](https://docs.npmjs.com/cli/v10/configuring-npm/package-json#bin)
- 根拠コミット: `045a8f4`（workflow_dispatchジョブ追加）、`5700d01`（基準画像更新）
- 確認日: 2026-08-08

## まとめ

修正コミットは`5700d01`である。
