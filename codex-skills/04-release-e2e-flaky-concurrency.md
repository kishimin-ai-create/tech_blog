# リリースE2Eのflakyを同時実行数から切り分ける

## 結論

リリースE2Eのflakyは、ブラウザや日本語入力ではなく、5つのブラウザプロジェクトが1つの画像生成インスタンスへ同時にリクエストを送ることで発生した。Glyph Forgeは1インスタンスあたり1件ずつ生成するため、待機中のリクエストがPlaywrightの30秒タイムアウトに達していた。

## 発生した問題

`RELEASE_E2E_BASE_URL=https://mojica.pages.dev/` を設定し、次のコマンドを実行した。

```powershell
bun run e2e e2e/tests/image-generation.large.test.ts
```

Playwrightの設定は5 workers、5 browser projectsである。ある実行では次の結果になった。

```text
Running 30 tests using 5 workers
5 failed
25 did not run
```

失敗したケースでは、`page.waitForEvent("download")` が30秒でタイムアウトし、画面には生成中のボタンが残っていた。

## 調査

### 直列実行との比較

同じテストを1 workerで実行すると、30件すべて成功した。

```text
Running 30 tests using 1 worker
30 passed (3.4m)
```

さらに1 workerで2回反復した場合も、60件すべて成功した。

```text
Running 60 tests using 1 worker
60 passed (5.6m)
```

一方、5 workersで3回反復すると、90件中3件が失敗した。

```text
Running 90 tests using 5 workers
3 failed
15 did not run
72 passed (3.8m)
```

この比較から、テスト操作そのものではなく、同時アクセス時の処理待ちが原因だと判断できる。

## 原因

Glyph Forgeの最新設定は、1インスタンスあたり同時処理1件、待機キュー9件、キュー待機30秒、画像生成30秒である。AppRunの最大スケール数が1の状態では、5ブラウザのリクエストが1インスタンスへ集中する。

`serial` は同じプロジェクト内のテストを直列化するが、異なるbrowser project間の実行は直列化しない。そのため、各ブラウザの最初のテストが同時に開始される。

## 解決方針

短期的な切り分けでは、1 workerで実行するとテストが作る同時負荷を除去できる。ただし、これは本番の処理能力を増やす対応ではない。

本番で5件程度の同時生成を扱うには、次のいずれかが必要になる。

* AppRunで複数のGlyph Forgeインスタンスを常時起動する
* Glyph Forgeの同時処理数を負荷試験の結果に基づいて増やす
* キュー待機時間とAPI・ブラウザのタイムアウトを、実際の最大待ち時間に合わせる

リトライだけを追加すると、容量不足を隠して重複生成を増やすため、根本対策にはならない。

## 検証結果

1回の成功ではflaky解消とは言えない。5 workersで複数回反復し、全ケースが成功することと、503・タイムアウト・メモリ不足が発生しないことを確認する必要がある。

## まとめ

* 5 workers時の失敗は、Glyph Forgeの1インスタンス同時処理1件による待機が原因だった。
* `workers=1`の成功は、同時実行競合を避けた結果であり、本番容量の証明ではない。
* AppRunのスケール設定、Glyph Forgeの処理能力、テストタイムアウトを同じ前提で設計する必要がある。

## 参考資料

* [FastAPI: Concurrency and async / await](https://fastapi.tiangolo.com/async/)
* [FastAPI: Deployment Concepts](https://fastapi.tiangolo.com/deployment/concepts/)
