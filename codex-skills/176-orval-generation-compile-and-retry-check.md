# Orvalの生成成功を、型チェックと再試行の動作確認につなげる

## 結論

生成コマンドの終了だけでは、生成されたhookをアプリケーションで使えるかまでは確認できない。profile-hubでは小さなOpenAPI fixtureで生成し、型チェックを通したうえで、実際の通信境界とつないで失敗からの再試行を確認した。

確認日は2026年10月27日。Orval 8.28.1、TanStack Query 5.102.8、Zod 4.5.4を使った。この記事は生成基盤の確認方法を扱い、本番APIの完成や接続成功を示すものではない。

## 本番契約と生成確認用の入力を分けた

生成設定は`OPENAPI_URL`を必須とし、React Query向けAxiosクライアントとZod validatorを生成する。小さなfixtureには`/__generation_smoke__`と、必須の文字列`message`を持つschemaを用意した。本番APIではないことをfixtureとREADMEに明記している。

通信先の仕様がまだ揃っていなくても、この入力で生成設定とアプリケーション側の接続点を確認できる。ただし本番schemaの網羅性や互換性は、この結果から判断できない。

## mutatorまでつなぐ

[Orvalのcustom Axiosガイド](https://orval.dev/docs/guides/custom-axios/)を参照し、生成コードが呼び出すmutatorを用意した。ここにbase URL、10秒のtimeout、`withCredentials: false`を置き、戻り値はAxiosのレスポンス全体ではなく`response.data`にする。

型が合うことに加えて、生成hookが期待する戻り値と、利用側が扱うデータが同じになることを確認する必要がある。共通の通信設定だけを読んで済ませず、実際に生成された呼び出しと合わせて調べた。

## 確認した順序

| 段階       | 実施内容                                                  | 確認できたこと                       |
| ---------- | --------------------------------------------------------- | ------------------------------------ |
| 生成       | fixtureを指定して`bun run api:generate`を2回実行          | 設定から生成処理が完了する           |
| コンパイル | `bun run typecheck --force`                               | 生成物とアプリ側の型が接続できる     |
| 実行       | 一時テストで生成hook・mutator・Provider・QueryStateを接続 | HTTP 503から再試行して成功表示になる |
| 入力検証   | 生成Zod schemaへ正常・異常入力を渡す                      | fixtureの検証条件が実行される        |

コマンドの実行場所は`apps/public-web`。fixtureのREADMEでは、生成物の混入を防ぐため使い捨てcheckoutでの確認を案内した。生成ファイルと一時的な結合テストはコミットしていないため、実行結果はこの時点の記録であり、継続的に同じ結合テストが走る構成ではない。

## 依存とwarningを分けて扱った

生成されたvalidatorが実行時に使うZodを直接依存にした。ただし、宣言する前にも生成と型チェックは成功していた。「不足したZod依存がコンパイルエラーを起こし、追加して直った」という経緯ではない。

Orval実行時には、解析に関する`import.meta`のwarningも出た。生成と型チェック、実際のViteビルドは成功したが、warningの解消を確認したとは書かない。

## 次に同じ基盤を確認するとき

まず小さな入力で生成設定を確認し、その後に本番の契約へ切り替える。生成、コンパイル、実行の結果を別々に残せば、どの境界で失敗したかを追える。本番入力へ切り替えた後は、そのschema固有のレスポンスやエラー処理も確認する必要がある。

## 根拠

- [PR #46：通信境界](https://github.com/kishimin-ai-create/profile-hub/pull/46)、[PR #50：生成確認](https://github.com/kishimin-ai-create/profile-hub/pull/50)
- [対象コミットのOrval設定](https://github.com/kishimin-ai-create/profile-hub/blob/4457f6f8624782d80bb8196a4c00c5319cf24325/apps/public-web/orval.config.ts)
- [生成fixtureの手順](https://github.com/kishimin-ai-create/profile-hub/blob/4457f6f8624782d80bb8196a4c00c5319cf24325/apps/public-web/src/tests/fixtures/README.md)
