# CIのE2Eダウンロードがタイムアウトした原因をAPIレート制限から切り分ける

## 結論

GitHub Actionsの画像生成E2Eでダウンロードイベントがタイムアウトした原因は、APIの起動失敗ではなく、CIジョブにMojica APIのレート制限設定がなかったことだった。Development環境では設定が未指定のまま許可数が0になり、`/images` が拒否されるため、ブラウザ側ではPNGダウンロードが発生しなかった。

## 発生した問題

### 症状

全5つのPlaywrightプロジェクトで、画像生成後のダウンロード待ちが失敗した。

```text
Error: page.waitForEvent: Test timeout of 30000ms exceeded.
waiting for event "download"
frontend/e2e/pages/image-generation-page.ts:45
frontend/e2e/pages/image-generation-page.ts:52
```

### 発生条件

- GitHub Actions `ubuntu-latest`
- ASP.NET Core `Development`
- Playwright 5プロジェクト
- 実APIとGlyph Forgeを使う画像生成E2E
- `RateLimit`設定なし

## 調査

最初の原因候補は、APIプロセスがhealth check前に終了していることだった。実際に別の実行では、`127.0.0.1:5063`への接続拒否が発生していたため、ビルド済みDLLを直接起動する修正を行った。

その後の実行ではAPIのhealth checkは通過し、テストはAPI呼び出し後のダウンロード待ちまで進んだ。そこでCI環境変数と設定バインドを確認した。

```text
RateLimit:PermitLimit = 0
RateLimit:Window = 00:00:00
```

`RateLimitOptionsValidator`は本番環境では起動時に検証されるが、Development環境では起動時検証を省略する。結果としてhealth checkは成功する一方、画像生成エンドポイントの固定ウィンドウ制限が有効なリクエストを許可しない状態になっていた。

## 原因

E2Eジョブの環境変数に次の設定がなかったことが原因だった。

- `RateLimit__PermitLimit`
- `RateLimit__Window`
- `RateLimit__QueueLimit`

この状態では、ブラウザの`waitForEvent("download")`が悪いのではなく、APIがPNGレスポンスを返していないため、ダウンロードイベント自体が発生しない。

## 解決方法

Push、Pull Request、Nightlyの各E2Eジョブへ、テスト専用の有効な設定を追加した。

```yaml
RateLimit__PermitLimit: 100
RateLimit__Window: 00:01:00
RateLimit__QueueLimit: 0
```

アプリケーションの通常の利用制限を無効化するのではなく、CIが検証したい画像生成フローをレート制限で遮らない値をジョブ単位で与えている。

## 動作確認

ローカルでは次の検証を実施した。

| コマンド | 結果 |
| --- | --- |
| `bun run typecheck` | pass |
| `bun run lint` | pass |
| `bun run test:small` | 130件 pass、Statements 96.64% |
| `bunx playwright test e2e/tests/image-generation.medium.test.ts --project="Google Chrome"` | 2件 pass |

修正コミットは`0cb3f02`で、PR #40のheadへpush済みである。GitHub Actionsの修正後実行結果は別途確認する必要がある。

## 再発防止

- Development環境でも、外部APIを使うE2Eジョブには必要な設定を明示する。
- health checkだけでなく、実際の代表リクエストが成功することをE2Eで確認する。
- ダウンロードイベントのタイムアウトを、ブラウザの問題と断定せず、直前のAPI応答・ステータスも確認する。

## まとめ

- 症状はブラウザのダウンロード待ちタイムアウトだった。
- API自体は起動していたが、レート制限の既定値0で画像生成を拒否していた。
- CIジョブへ明示的な`RateLimit`設定を追加して、E2Eの検証対象を実行可能にした。

