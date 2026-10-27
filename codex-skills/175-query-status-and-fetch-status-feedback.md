# TanStack QueryのstatusとfetchStatusを分けて読み込み状態を表示する

## 結論

初回読み込みとバックグラウンド更新を同じ画面にすると、取得済みの内容まで消してしまう。profile-hubでは`status`でデータの状態を、`fetchStatus`で取得処理の状態を判断し、成功済みの内容を表示したまま更新中であることを伝えた。

Reactの通信状態表示を作る人向けの実装記録である。確認日は2026年10月27日、対象はTanStack Query 5.102.8。対象PRは未マージである。

## 最初に整理した状態

[公式のQueriesガイド](https://tanstack.com/query/latest/docs/framework/react/guides/queries)でも、データの状態と取得処理の状態は別に扱われる。実装では次の組み合わせを表示の契約にした。

| status  | fetchStatus   | 表示                         |
| ------- | ------------- | ---------------------------- |
| pending | fetching      | 初回の読み込み中             |
| pending | idle / paused | データを待っている状態       |
| error   | 任意          | 汎用エラーと再試行ボタン     |
| success | fetching      | 取得済みの内容と更新中の案内 |
| success | idle          | 取得済みの内容               |

`pending`だけを見て「通信中」と断定しないことが要点だった。取得を実行していない場合にも同じデータ状態になり得るため、待機用の表示を分けた。

## 再試行を共通コンポーネントの契約にした

`QueryState`はQueryの結果全体に依存せず、表示判断に必要な`status`、`fetchStatus`と、再試行用の`refetch`を受け取る。成功時の内容はchildrenとして渡す。

エラー時には利用者向けの定型文を表示し、例外のmessageや通信先などの内部情報をそのまま出さない。再試行中はボタンを無効にする。これは、再取得が続いている間に同じ操作を繰り返せないようにするためである。

QueryClientはProvider内の遅延初期化で一度生成する。共通設定は`staleTime: 60_000`と`retry: false`にした。自動再試行をしない設定でも、ボタンから明示的に再取得できる。認証状態や永続化をこのProviderに追加したわけではない。

## 何を確認したか

コンポーネントテストでは、読み込み、待機、失敗からの再試行、内部診断情報の非表示、更新中の内容維持と重複操作の抑止を確認した。Storybookには通信を起こさずに状態を見られる6つのstoryを用意し、ビルド済みの画面もブラウザで確認した。

さらに一時的な結合テストでは、生成されたhookと実際のmutator、Provider、状態表示を接続した。HTTP 503からボタンで再試行し、成功した内容が表示されることを確認した。この一時テストと生成物はPRの成果物には含めていない。

## 適用範囲

今回の表示は初回エラーと明示的な再試行を中心にした基盤である。バックグラウンド更新が失敗しても古い内容を残す設計や、オフライン復帰時の詳しい案内は、各画面の要件に応じて別途判断する。

状態を増やす前に、「データがあるか」と「今取得しているか」を別々に書き出すと、表示を消す条件と補足案内だけを出す条件を決めやすい。

## 根拠

- [PR #47：Provider](https://github.com/kishimin-ai-create/profile-hub/pull/47)、[PR #48：状態表示](https://github.com/kishimin-ai-create/profile-hub/pull/48)、[PR #49：story](https://github.com/kishimin-ai-create/profile-hub/pull/49)
- [対象コミットのQueryState](https://github.com/kishimin-ai-create/profile-hub/blob/4457f6f8624782d80bb8196a4c00c5319cf24325/apps/public-web/src/components/QueryState.tsx)
- [対象コミットのProvider](https://github.com/kishimin-ai-create/profile-hub/blob/4457f6f8624782d80bb8196a4c00c5319cf24325/apps/public-web/src/providers/PublicQueryProvider.tsx)
