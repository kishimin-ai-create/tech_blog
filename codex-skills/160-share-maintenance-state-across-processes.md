# APIと収集コマンドが別プロセスなら、停止状態を共有DBへ保存する

## 結論

slope-collectorの停止状態はAPIプロセスのメモリに置かず、収集データと同じDBの専用テーブルへ保存した。API、日次収集、バックフィル、復旧確認コマンドが同じ状態を読み取るためである。

対象は、Web APIとバッチ処理を別プロセスで動かす構成である。2026年10月24日にWindows・Python 3.14.7で検証した。

## 背景

今回のGUIは別アプリとして後から作る。公開GUI側のサーバーが利用者を認証し、内部接続でslope-collectorの管理APIを呼ぶ方針にした。slope-collector内にGUIやログイン処理を追加したわけではない。

一方、収集処理はFastAPIに埋め込まれておらず、`collect-daily`などの独立したコマンドで動く。APIが受け取った停止操作を、別プロセスにも伝える必要があった。

## 解決したい課題と制約

プロセスを再起動しても、停止状態、再開方法、確認間隔を保持する。公式サイトとアーカイブは、収集元ごとに別の制御対象にする。記事の保存状態とメンテナンス設定を混同しない。

## 検討した選択肢

### Option A：APIのメモリ内で保持する

単一プロセス内の状態参照は容易だが、今回の独立したコマンドや再起動後の状態共有には足りない。

### Option B：専用テーブルへ保存する

既存DBを共有し、各プロセスが読み直す。マイグレーションとDBアクセスが必要になるが、今回必要な永続化と共有を同じ仕組みで扱える。

| 評価軸 | API内のメモリ | 専用DBテーブル |
| --- | --- | --- |
| 別プロセスからの参照 | 別の共有手段が必要 | 同じDBを参照 |
| 再起動後の保持 | できない | できる |
| 追加の運用 | プロセス内の管理 | スキーマ移行が必要 |

## 最終判断と実装

`collection_controls`テーブルを追加した。キーは`source_a:official`などの収集元とルートの組み合わせであり、メンバー単位ではない。状態には次の項目を持たせた。

| 項目 | 用途 |
| --- | --- |
| `paused`、`resume_mode` | 停止中か、手動再開か自動再開か |
| `check_interval_minutes`、`reason` | 確認間隔と任意の停止理由 |
| `next_check_at`、`last_checked_at` | 次の確認予定と最後の確認時刻 |
| `last_check_succeeded`、`last_check_error` | 前回の確認結果 |
| `revision` | 古い復旧結果を判別する識別子 |

行が存在しない対象は通常稼働として扱う。ただしテーブルへ接続できない場合まで通常稼働にすると、停止設定を無視することになる。保存先の障害は`ControlStorageError`へ変換し、APIでは詳細を伏せた503を返す。収集側も停止確認を迂回して通信を続けない。

## 実装後の結果

同じSQLiteのセッションファクトリーを使ってAPIアプリを作り直し、停止状態が残ることを確認した。開発・本番の設定を切り替えても管理APIを利用でき、公式を止めてもアーカイブの状態は変わらなかった。

マイグレーションのテストでは、既存のsource行を残したまま制御テーブルを作成・削除できた。テーブルがない場合のAPI応答も検証している。

## トレードオフ・今後の懸念

要求ごとに共有状態を読み直すためDBアクセスが増えるが、その負荷は測定していない。日時はUTCとして扱う。実際のMySQLでの移行と競合動作、本番の接続境界は未検証である。

管理APIは既存の内部APIと同じく認証を実装していない。公開GUI側の認証と、サーバーからだけ到達できる配置が必要になる。コードをGitHubへ置くこと、デプロイ、DB移行、タイマー有効化は別の作業である。

## 根拠と関連記事

- [永続化モデル](https://github.com/kishimin/slope-collector/blob/cfd98c8aa1dd14e88c7b48791ed271f795d8dd80/app/models/collection.py)、[制御Repository](https://github.com/kishimin/slope-collector/blob/cfd98c8aa1dd14e88c7b48791ed271f795d8dd80/app/repositories/control.py)
- [再起動後のAPI状態と保存先障害のテスト](https://github.com/kishimin/slope-collector/blob/cfd98c8aa1dd14e88c7b48791ed271f795d8dd80/tests/medium/test_collection_control.py)
- [スキーマ移行の回帰テスト](https://github.com/kishimin/slope-collector/blob/cfd98c8aa1dd14e88c7b48791ed271f795d8dd80/tests/medium/test_collection_migration.py)
- [古い復旧結果を無効にするrevision](./162-reject-stale-recovery-results.md)
