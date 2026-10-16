---
title: "E2Eをコンテナへ移すときにAPI接続とレポート回収を揃える"
tags: [Docker, Playwright, GitHubActions, E2E]
---

# E2Eをコンテナへ移すときにAPI接続とレポート回収を揃える

## はじめに

ブラウザE2EをホストからDockerへ移すと、APIの接続先、フロントエンドのビルド時設定、成果物の保存先が変わる。MojicaのCI修正では、描画環境の固定に加えて、この3つを揃えた。

この記事はコンテナとホストの間の受け渡しを扱う。描画環境を固定した背景は[日本語VRTの記事](./114-japanese-vrt-rendering-environment.md)に分ける。

## 前提・環境

- CI設定: GitHub ActionsのLinuxランナー。APIはホストの`127.0.0.1:5063`、Glyph Forgeは別コンテナで起動する。
- E2E: Docker内でVite preview、Storybook、Playwrightを起動する。
- フロントエンド: `import.meta.env.VITE_API_URL`をAPIクライアントに使用する。
- 手元の検証: Windows上のDocker Desktop Linuxコンテナ。APIの公開ポートは15063。

## やってみた結果

API接続先を渡してコンテナ内でビルドし、更新済み基準画像に対するE2Eを実行した。45件成功・30件スキップだった。停止済みコンテナからHTMLレポートとtest-resultsもコピーできた。

GitHub Actions側のLinux host networkingはworkflowの設定を確認した段階であり、このローカル成功だけでその経路の成功まで保証しない。

## 実装・検証

### Step 1: APIのURLをビルド前に渡す

ホスト側でbuildした成果物をコンテナへコピーする構成では、ビルド時のAPI URLが残る可能性がある。今回は環境変数を渡してからコンテナ内でbuildする構成にした。

Dockerfileの既定コマンドは次のとおりである。

```dockerfile
CMD ["bash", "-c", "bun run build && bun run e2e"]
```

CIのLinuxランナーでは、E2Eコンテナからホスト側APIへ接続するためhost networkingを指定した。

```yaml
run: |
  docker run --name mojica-e2e --init --ipc=host --network host \
    --env VITE_API_URL mojica-e2e
```

`--env VITE_API_URL`はジョブの環境変数をコンテナへ渡す。ジョブでは`http://127.0.0.1:5063`を設定している。これにより、既定コマンドのbuildがその値を使用する。

手元では別の経路を使って検証した。

```powershell
docker run --name mojica-e2e-final --init --ipc=host --env VITE_API_URL=http://host.docker.internal:15063 mojica-e2e:verified
```

`host.docker.internal`を使用した結果と、Linuxの`--network host`を同一の検証として扱わないことが、今回の報告上の注意点だった。

### Step 2: ヘルスチェックの後に実際の生成を試す

ローカルAPIは起動して`/health`へ応答できても、画像生成時に`OptionsValidationException`で失敗した。最初はレート制限のWindowが不足していた。さらに、Glyph Forgeの通信設定も補う必要があった。

検証用APIを再作成する際に、CIと同様に次の設定を明示した。

```text
RateLimit__PermitLimit=100
RateLimit__Window=00:01:00
RateLimit__QueueLimit=0
GlyphForge__Timeout=00:01:00
```

Glyph ForgeのURLとCORSの許可Originもローカルの接続経路へ合わせた。その後、`POST /images`が200、Content-Typeが`image/png`、レスポンスが5394 bytesであることを確認した。これは検証用APIの設定不足への対応であり、GitHubで起きた画像比較失敗の原因とは分けている。

### Step 3: 削除する前にレポートを回収する

テスト終了と同時に`--rm`でコンテナを削除すると、内部のレポートも取り出せなくなる。CIでは名前付きコンテナを残し、次のステップでコピーする構成にした。

```yaml
- name: Collect E2E reports
  if: ${{ !cancelled() }}
  run: |
    docker cp mojica-e2e:/app/playwright-report frontend/playwright-report
    docker cp mojica-e2e:/app/test-results frontend/test-results
```

この後、既存のartifact uploadを実行し、最後のcleanupでコンテナを削除する。テスト失敗時にも回収ステップが動くよう条件を指定している。なお、build失敗などでレポートそのものが生成されなかった場合、コピーも失敗し得る。

手元では、E2Eが終了した後のコンテナからHTMLレポートとtest-resultsを一時保存先へ回収できた。検証用コンテナは回収後に削除した。

## つまずいたところ

### APIの起動成功を画像生成の成功として扱えなかった

問題はブラウザの起動前ではなく、画像生成リクエストが通る経路に必要な設定だった。起動ログ、`/health`、実際の`POST /images`を別の確認として扱うことで、ローカルAPIの設定不足をブラウザやVRTの問題へ混ぜずに切り分けた。

## 学んだこと

E2Eのコンテナ化では、イメージ内だけで完結する設定と、ホストへ接続する設定を分けて確認する必要があった。今回の実装では、build時のAPI URL、ネットワーク経路、実リクエスト、停止後の成果物回収までを確認項目にした。

## まとめ

今回ローカルで確認できたのは、Docker DesktopのLinuxコンテナからローカルAPIを使い、画像生成を含むE2Eが成功し、停止後のレポートを回収できることまでである。GitHub ActionsのLinux host networkingとartifact uploadを含む一連の実行結果は、PR公開後に別途確認する。

## 参考資料

- [環境固定・CI変更のコミット](https://github.com/kishimin/mojica/commit/466a4bd)
- [再現手順](https://github.com/kishimin/mojica/blob/abcad1b/frontend/e2e/README.md)
- [Playwright Docker公式資料](https://playwright.dev/docs/docker)

確認日: 2026年10月16日。
