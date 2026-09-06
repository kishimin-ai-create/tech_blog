# APIの保護を維持したままPlaywright E2Eを5並列で実行する

## 対象読者

実APIを使うPlaywright E2EをCIで実行しており、テスト時間を短縮しながらAPIのレート制限も維持したい開発者を対象にする。

## スコープ

MojicaでPlaywrightのworker数を1から5へ変更し、Development環境のAPIレート制限を明示して並列実行を支えた変更を扱う。CI runnerの性能比較や、実運用環境のレート制限値の決定方法は扱わない。

## 結論

実APIを使うE2Eの並列度を5へ上げる場合でも、レート制限を無効化するのではなく、E2E環境に明示的な上限を設定する。MojicaではPlaywrightの`workers`を5に変更し、Development環境の画像生成APIを1分あたり100件、キューなしで受け付ける設定を追加した。設定値はバックエンドのテストでも確認する。

## 背景

E2EはMojica APIとGlyph Forgeを実際に起動して画像生成を行うため、workerを増やすとAPIへの同時要求も増える。並列度だけを変更すると、テストが上流サービスの保護機構に衝突し、テスト対象の機能ではなくレート制限の結果で失敗する可能性がある。一方、レート制限を外すと、E2Eのために本番に近い保護を失わせることになる。

## 変更内容

### Playwrightの並列度を5へ変更する

`frontend/playwright.config.ts`のworker数を次のように変更した。

```ts
export default defineConfig({
  fullyParallel: false,
  workers: 5,
});
```

`fullyParallel`は変更していない。worker数の変更と、テストケースを完全に並列化する設定は別の判断なので、今回の変更範囲をworker数に限定している。

### Development環境にレート制限を設定する

`backend/Mojica.Api/appsettings.Development.json`へ次の設定を追加した。

```json
{
  "RateLimit": {
    "PermitLimit": 100,
    "Window": "00:01:00",
    "QueueLimit": 0
  }
}
```

同じ値はCIのE2Eジョブにも環境変数として設定している。ローカルのDevelopment起動とCI起動で、異なる保護条件を暗黙に作らないためである。

### 設定をテストで固定する

バックエンドの`RateLimitRegistrationMediumTests`では、Development環境でAPIが起動できることに加え、解決された`RateLimitOptions`の値を確認する。

```csharp
Assert.Equal(100, options.PermitLimit);
Assert.Equal(TimeSpan.FromMinutes(1), options.Window);
Assert.Equal(0, options.QueueLimit);
```

Production環境では設定不足を起動時に検出する既存テストも残している。Developmentにはテスト可能な既定値を用意し、Productionには未設定を許さないという役割分担である。

## 検証

変更後、MojicaのE2Eを実行し、Google Chrome、Microsoft Edge、Desktop WebKit、Android Chrome、iPhone Safariを含む45件が成功した。また、RateLimit設定のバックエンドテスト、型チェック、Lint、フォーマット確認を実行した。

この結果は、確認したローカル環境での観測結果である。CI runnerでの実行時間や、同じ5 worker設定を長期運用した場合の上限余裕までは、この変更だけでは確定できない。

## 事実・判断・未確認事項

- FACT: `frontend/playwright.config.ts`の`workers`は1から5へ変更された。
- FACT: Development環境の`PermitLimit`は100、`Window`は1分、`QueueLimit`は0として定義された。
- FACT: CIのE2Eジョブにも同じレート制限値が設定されている。
- FACT: 設定値を`IOptions<RateLimitOptions>`から読み取り、バックエンドテストで確認している。
- FACT: ローカルの5ブラウザE2Eは45件成功した。
- INFERENCE: 並列度とサービスの保護条件を同じ変更単位で管理すると、E2Eの負荷条件を追跡しやすい。
- ASSUMPTION: 100件/分が将来のE2E件数とCI実行頻度に対して十分かは、運用データを見て再評価する必要がある。

## トレードオフ・今後の懸念

workerを増やすと実行時間短縮が期待できる一方、サービスへの同時負荷が増える。100件/分という値はE2E環境のための設定であり、Productionの適切な値を意味しない。また、複数のCI実行が同じ共有サービスへ向く構成へ変わる場合は、1ジョブ内のworker数だけでなく、ジョブ間の合計要求数も評価しなければならない。

## まとめ

実APIを使うE2Eの並列化では、Playwrightのworker数だけを増やすと不安定化しやすい。worker数、APIのレート制限、CIの環境変数、バックエンド設定テストを同じ契約として管理することで、保護機構を残したまま並列実行を検証できる。

## 参考

- `mojica/frontend/playwright.config.ts`
- `mojica/backend/Mojica.Api/appsettings.Development.json`
- `mojica/backend/Mojica.Api.Tests/Infrastructure/RateLimitRegistrationMediumTests.cs`
- `mojica/.github/workflows/ci-push.yml`
- `mojica/.github/workflows/ci-pull-request.yml`
- `mojica/.github/workflows/ci-nightly.yml`
- Mojica commit `0cb3f02 fix: configure API rate limits for E2E`
- Mojica commit `a030e09 test: require development rate limit defaults`
- Mojica commit `465ced3 feat: allow five parallel E2E workers`

確認日: 2026-09-06
