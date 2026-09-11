# Retry-Afterのカウントダウンをコールバック回数ではなく時刻で計算する

## 結論

Mojicaの再試行待ち時間は、`setInterval`のコールバック回数を信頼せず、開始時に固定したdeadlineと現在時刻の差から毎回計算するようにした。これにより、タイマーコールバックの遅延があっても経過時間を基準に残り秒数を表示できる。

## 背景

`useRetryAfterCountdown`は429応答の`Retry-After`秒数を画面に表示する。単純に1秒ごとにカウンターを減らす実装では、ブラウザーやテスト環境でコールバックが遅延したとき、実際の経過時間と表示がずれる可能性がある。

## 採用した実装

開始時にdeadlineを一度だけ計算し、各tickで現在時刻との差を切り上げる。

```ts
const deadline = Date.now() + retryAfterSeconds * 1000;
const timerId = setInterval(() => {
  const nextRemainingSeconds = Math.max(
    0,
    Math.ceil((deadline - Date.now()) / 1000),
  );

  setRemainingSeconds(nextRemainingSeconds);
}, 1000);
```

deadlineを各tickで再計算しないのは、待ち時間の起点を固定するためである。コメントにもこの理由を残している。

## 検証

Smallテストでは通常の1秒経過に加え、システム時刻を2秒進めた場合に残り時間が3秒になることを確認した。これにより、intervalの呼び出し回数ではなく経過時刻を使っていることを検証できる。

関連コミット:

- `3f94024 test: cover elapsed retry time`
- `c4216f5 fix: base retry countdown on elapsed time`
- `a1357e5 docs: explain fixed retry deadline`

## トレードオフ

この実装は壁時計の変化を基準にするため、端末時刻が大きく変更された場合も表示へ影響する。一方、タイマーの実行間隔を正確に保つことを前提にしないため、ブラウザーのタイマー遅延を単純なtickカウントより正しく扱える。

## まとめ

再試行待ち時間のような経過時間は、コールバック回数ではなく固定deadlineとの差から導出する。テストでは時刻を進めて検証すると、実装の意図をユーザーが観測する振る舞いとして表現できる。
