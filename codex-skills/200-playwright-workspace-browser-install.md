# CI の Playwright がブラウザーを見つけられなかった原因

## 結論

ワークスペース構成の CI で、ブラウザーのインストールをリポジトリ直下から行い、テストを別のワークスペースから実行していた。インストールを各ワークスペースで実行するよう変更したところ、管理画面の Chromium と WebKit の E2E が成功した。

## 発生した問題

GitHub Actions の管理画面 E2E は、両ブラウザーで `browserType.launch` 時に実行ファイルが存在しないと報告した。ブラウザーのキャッシュがヒットしたため、当初の設定ではインストール自体を省略していた。

## 調査と対応

まずキャッシュの有無にかかわらず `playwright install chromium webkit` を実行するようにしたが、再実行でも同じファイル不足が起きた。さらに確認すると、インストールはリポジトリ直下、テストは `apps/admin-web` で実行していた。Playwright の依存関係はワークスペースにあり、同じ実行場所のコマンドで必要なリビジョンを揃える必要があった。

次のように、インストールとテストの作業ディレクトリを揃えた。OS のライブラリもキャッシュに含まれないため、システム依存の導入は毎回実行する。

```yaml
- name: Install Playwright browsers
  run: bunx playwright install chromium webkit
  working-directory: apps/${{ matrix.workspace }}
```

## 動作確認

修正後、手動実行した夜間 CI で管理画面の E2E と VRT を含むブラウザージョブが成功した。別の実行でもブラウザージョブの成功を確認した。一方、後者では API の関数カバレッジが閾値未達であり、ワークフロー全体は失敗している。ブラウザージョブの成功と CI 全体の成功は区別する。

## 学び

キャッシュがヒットしたことは、実際に使う Playwright のバージョンとブラウザーのリビジョンが揃っている証拠にはならない。依存関係がワークスペースごとに解決される場合は、インストールと実行のディレクトリを揃え、実際の CI で両方のブラウザーを起動して確かめる。

根拠: [夜間ブラウザーテストの PR](https://github.com/kishimin-ai-create/profile-hub/pull/145)、[成功した夜間 CI](https://github.com/kishimin-ai-create/profile-hub/actions/runs/41265513173)、[API カバレッジが失敗した後続実行](https://github.com/kishimin-ai-create/profile-hub/actions/runs/41265823241)。
