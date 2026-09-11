# Playwright E2Eを既存のテストサイズ別CIへ統合する

## 対象読者

GitHub ActionsでフロントエンドのブラウザE2Eを運用しており、E2E専用ワークフローを増やさず既存のテスト実行計画へ組み込みたい開発者を対象にする。

## スコープ

Playwright E2EをSmall・Medium・Largeの既存ワークフローへ統合する方法を扱う。テストケースをサイズごとに再分類する方法や、E2Eのアプリケーション仕様そのものは扱わない。

## 結論

Mojicaでは、E2E専用のCIワークフローを残さず、既存のイベント別ワークフローへE2Eジョブを追加した。pushではSmall、Pull RequestではMedium、nightlyではLargeという既存の入口に対応させ、それぞれのジョブでAPI・画像生成サービス・フロントエンドを起動してからPlaywrightを実行する構成にした。

## 背景

当初はE2Eを独立した`ci-e2e.yml`で実行していた。しかし、リポジトリではテストサイズごとにpush、Pull Request、nightlyのワークフローがすでに存在していた。E2Eだけ別の実行経路にすると、どの変更でどの検証が走るのかが分散するため、既存のワークフローへ取り込む方針に変更した。

## 実装

### イベントごとのジョブへ統合する

次の3ファイルへE2Eジョブを追加した。

| ワークフロー | ジョブ | 実行タイミング |
| --- | --- | --- |
| `.github/workflows/ci-push.yml` | `frontend-e2e-small` | push |
| `.github/workflows/ci-pull-request.yml` | `frontend-e2e-medium` | Pull Request |
| `.github/workflows/ci-nightly.yml` | `frontend-e2e-large` | nightly |

現時点でPlaywrightのテストファイルはMedium分類であるため、ジョブ名はサイズを表すが、実行コマンドは3つとも`bun run e2e`である。将来SmallまたはLargeのE2Eを追加した場合は、ファイル指定を調整して実行範囲を分離できる。

### CI上で依存サービスを起動する

ブラウザだけを起動しても、画像生成E2Eは成立しない。CIジョブでは次の順序で依存サービスを準備する。

1. `kishimin/glyph-forge`を固定コミットでcheckoutする。
2. Dockerイメージをbuildし、8080番ポートで起動する。
3. `/health`へのリクエストが成功するまで待機する。
4. Mojicaの.NET APIをrestore・buildし、5063番ポートで起動する。
5. Mojica APIの`/health`が成功するまで待機する。
6. フロントエンドをbuildしてPlaywrightを実行する。

サービスの準備完了を固定時間のsleepで待たず、health endpointへのリトライで確認する点が重要である。起動が遅い実行環境でも、サービスが利用可能になった時点で次へ進める。

### 失敗時の調査材料を保存する

E2E実行後は、成功・失敗にかかわらず次のArtifactを保存する。

- `frontend/playwright-report`
- `frontend/test-results`
- `mojica-api.log`

ログが存在しない場合でもArtifact保存ステップ自体で二次的に失敗しないよう、`if-no-files-found: ignore`を指定している。サービスは最後に停止し、テスト失敗時も後片付けが走るようにしている。

## 検証

変更後、ローカル環境で`bun run e2e`を実行し、5つのPlaywrightプロジェクトを含む45件が成功した。対象プロジェクトはGoogle Chrome、Microsoft Edge、Desktop WebKit、Android Chrome、iPhone Safariである。

CI設定に対しては、ワークフローの差分確認、Prettier、`git diff --check`、フロントエンドのbuild、型チェック、Lintを実行した。GitHub Actions上での実行結果は、push前のローカル検証だけでは確認できないため、別途CI実行が必要である。

## 事実・判断・未確認事項

- FACT: E2E専用ワークフローを削除し、push・Pull Request・nightlyの既存ワークフローへジョブを追加した。
- FACT: 各ジョブはGlyph ForgeとMojica APIをhealth endpointで確認してからPlaywrightを実行する。
- FACT: ローカルの`bun run e2e`は45件成功した。
- INFERENCE: 実行イベントとテストサイズの入口を揃えることで、CIの実行場所を追跡しやすくなる。
- ASSUMPTION: GitHub Actionsのrunner上でも、固定したGlyph ForgeコミットのDocker buildと5ブラウザのインストールが制限時間内に完了するかは、CI実行で確認する必要がある。

## トレードオフ・今後の懸念

同じE2Eスイートを3つのイベントで実行するため、現時点ではSmall・Medium・Largeの実行内容が実質的に重複している。ジョブを分けたことで入口は整理できたが、テストファイルをサイズ別に絞り込む作業は残っている。また、APIとGlyph Forgeを毎回buildするため、キャッシュを使わない場合はCI時間が長くなる可能性がある。実際のCI実行時間と失敗時のArtifactを確認してから、キャッシュや実行対象の分割を検討する。

## まとめ

E2Eを既存CIへ統合するときは、単に`playwright test`を追加するだけでなく、テストの入口、依存サービスの起動、health check、失敗時のArtifact保存までを1つのジョブとして設計する必要がある。MojicaではE2E専用ワークフローを削除し、既存のSmall・Medium・Largeワークフローへ段階的に組み込んだ。

## 参考

- `mojica/.github/workflows/ci-push.yml`
- `mojica/.github/workflows/ci-pull-request.yml`
- `mojica/.github/workflows/ci-nightly.yml`
- `mojica/frontend/playwright.config.ts`
- Mojica commit `17d6850 ci: integrate e2e tests with sized workflows`
- Mojica commit `633e7f4 style: format frontend files with prettier`

確認日: 2026-09-06
