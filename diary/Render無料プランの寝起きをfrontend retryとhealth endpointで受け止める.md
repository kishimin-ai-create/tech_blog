# Render無料プランの寝起きをfrontend retryとhealth endpointで受け止める

## はじめに

一方で、400 などの validation error は retry しません。ユーザー入力やリクエスト内容が原因のエラーを繰り返しても復旧しないためです。

## 対象読者

- Render の無料プランで backend を動かしている人
- frontend からの API 呼び出しで 502 / 503 / 504 が出るケースに向き合っている人
- UptimeRobot などで監視しながら、無料運用の体験を少しでも安定させたい人

## この記事で扱うこと

この記事では、Render 無料プランで backend がスリープする前提を受け入れたうえで、つづる日記に入れた2つの対策をまとめます。

- frontend 側で一時的な backend wake-up failure を retry する
- backend 側に DB 非依存の `/health` endpoint を追加する

「無料プランでも絶対に寝ないようにする」方法ではありません。無料プランでは backend が寝ることを前提に、寝起き中の失敗をユーザー体験と監視で受け止めるための対応です。

## 背景

Render の無料 web service は、しばらくアクセスがないとスリープします。次のアクセスで起動しますが、その起動中に frontend が API を呼ぶと、一時的に 502 / 503 / 504 やネットワークエラーになることがあります。

つづる日記では Next.js frontend が `/api/...` を呼び、その route proxy が Render 上の backend に転送します。backend が寝ている瞬間に一覧取得や作成処理が走ると、ユーザーには「壊れている」ように見えてしまいます。

そこで、無料運用のまま次の2段構えにしました。

1. frontend が一時的な失敗を自動 retry する
2. UptimeRobot の監視先として軽量な `/health` を用意する

## frontend retry

Orval が生成した API client は `frontend/app/api/mutator/custom-instance.ts` の `customInstance` を通ります。

ここに retry を追加しました。対象は次のような、Render backend の寝起きで起こりやすい一時的な失敗です。

- 502
- 503
- 504
- 一時的なネットワーク失敗

一方で、400 などの validation error は retry しません。ユーザー入力やリクエスト内容が原因のエラーを繰り返しても復旧しないためです。

デフォルトでは `3秒 × 最大20回` 待つようにしました。最大で約1分待てるため、無料 backend の起動待ちを吸収しやすくなります。

テストでは、最初のリクエストが 503 で失敗し、次のリクエストで成功するケースを追加しました。また、400 は retry しないことも確認しています。

対象ファイル:

- `frontend/app/api/mutator/custom-instance.ts`
- `frontend/app/api/mutator/custom-instance.small.test.ts`

## backend health endpoint

UptimeRobot の監視先として `/openapi.json` を使うこともできます。ただ、監視用途ならもっと軽い endpoint のほうが向いています。

そこで backend に `GET /health` を追加しました。

```json
{ "status": "ok" }
```

この endpoint は DB や repository に依存しません。さらに、startup migration が pending の状態でも `/health` は 200 を返すようにテストしています。

これにより、UptimeRobot の監視先は次のようにできます。

```text
https://diary-lt6c.onrender.com/health
```

対象ファイル:

- `backend/src/app.ts`
- `backend/src/app.small.test.ts`
- `backend/src/server.small.test.ts`

## 確認したこと

frontend 側では次のコマンドが通ることを確認しました。

```bash
bun run typecheck
bun run lint
bun run test
```

backend 側でも次のコマンドが通ることを確認しました。

```bash
bun run typecheck
bun run lint
bun run test
```

backend lint では既存の `console` warning が出ますが、エラーはなく成功しています。

## 注意点

この対応は Render 無料プランの制約をなくすものではありません。

backend が寝ること自体を確実に避けたいなら、有料プランにする必要があります。今回の対応は、無料運用のまま「寝起きで一時的に失敗する」問題を、retry と監視しやすい endpoint で受け止めるためのものです。

## まとめ

Render 無料プランでは、backend が寝ることを前提に設計したほうが現実的です。

つづる日記では、frontend の API client に一時エラー retry を入れ、backend に DB 非依存の `/health` を追加しました。これで、ユーザー操作時の寝起き失敗を待ちやすくしつつ、UptimeRobot で軽量に監視できる形にしました。
