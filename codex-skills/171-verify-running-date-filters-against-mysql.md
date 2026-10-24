# 日付条件を持つ稼働APIの結果を、実MySQLのID列と順序まで照合する

## はじめに

対象読者は、DBを使うAPIの変更をローカルのDocker環境へ反映し、実際の応答まで確認したい開発者である。2026年10月24日のslope-collectorでは、著者別の全件取得へ任意の日付条件を加えた後、稼働APIの4通りの応答を既存MySQLへの読み取りと照合した。

確認したいのは「コンテナが起動したか」に加えて、「新しい条件で、保存済みの対象データを返しているか」である。実際の著者、記事ID、本文、取得元、記事件数はこの記事に掲載しない。

## 前提・環境

- Windows上のDocker DesktopとCompose。APIとMySQLは別サービス。
- 開発用APIはホストの`127.0.0.1:8080`へ公開する。
- APIから既存のMySQLへ接続する。DBへの書き込み、マイグレーション、外部サイトの収集は行わない。
- 対象は`GET /entities/{entity_id}/records`の日付なし、`from`のみ、`to`のみ、両端あり。

本記事は作業時の実行結果を記録したもので、記事の執筆時に同じ操作を再実行したものではない。API契約の設計は[記事170](./170-integrate-optional-dates-into-entity-records.md)で扱う。

## やってみた結果

代表として選んだ1著者について、4モードともHTTP 200で応答し、返されたID列は同じ著者に対するSQLの期待ID列と順序まで一致した。応答の各記事が`id`・`title`・`body`の3項目であることも確認した。

APIとDBはhealthyとなり、更新前後でDBコンテナIDと`mysql-data`のマウントは同じだった。確認できたのはAPIの更新と既存DBを使った読み取りであり、DB全件の内容が完全に不変であることを照合したわけではない。

## 実装・検証

### APIを更新してから稼働経路を調べる

以下は、リポジトリルートで実行したサービス操作である。既存の開発用設定とDBの起動を前提とし、APIの再作成による一時停止を伴う。

```powershell
docker compose --env-file .env.development config --quiet
docker compose --env-file .env.development build api
docker compose --env-file .env.development up -d --no-deps --wait --wait-timeout 60 api
```

APIのみを再作成し、DBコンテナとそのマウントを更新前後で確認した。イメージ作成と稼働コンテナへの適用を分ける考え方は[既存記事143](./143-apply-verified-uv-image-without-recreating-db.md)で説明している。

### 起動確認と契約確認を分ける

検証用PythonをAPIコンテナ内の`/app/.venv/bin/python`で実行し、`urllib.request`で起動中のAPIへ要求した。このときの`http://127.0.0.1:8080`は、APIコンテナ自身のループバックである。ホストからのアクセス確認とは実行場所が異なる。

| 確認対象 | 観測した結果 |
| --- | --- |
| `/health` | HTTP 200、`status=ok` |
| `/openapi.json` | 旧`/records/by-date`ルートなし |
| 著者別ルートの定義 | `from`と`to`がそれぞれ任意 |
| 各記事の応答項目 | `id`・`title`・`body`のみ |

ヘルスチェックは生存確認となる。一方、旧ルートの削除と任意パラメーターの確認は、新しい契約が稼働APIへ反映されていることを調べる別の確認である。

### APIと独立した時刻値で期待結果を求める

既存設定からSQLAlchemyのセッションを作り、著者IDを1つ読み取った。その同じ著者をAPIとSQLの両方へ指定した。

SQL側ではAPI内部のクエリ構築関数を呼ばず、`Record.id`への条件を組み立てた。日付の変換結果を明示したUTC値として置き、指定した側の条件だけを加えた。

| APIの日付指定 | SQLに追加したUTC条件 |
| --- | --- |
| 日付なし | 日付条件なし |
| `from=2026-10-01` | `published_at >= 2026-09-30 15:00:00+00:00` |
| `to=2026-10-07` | `published_at <= 2026-10-07 14:59:59.999999+00:00` |
| 両端あり | 上の2条件 |

すべてのSQLに同じ`entity_id`条件を付け、`published_at DESC, id DESC`で並べた。この方法は、API側の変換関数をそのまま期待値生成にも使うことを避け、今回の日付に対して独立した期待境界を用意するものだった。

### 集合だけでなく列として比べる

次は実際に行った比較の最小表現である。`actual_ids`はHTTP応答の順序のまま取り出したID列、`expected_ids`はSQLの並び順のまま取得したID列を表す。

```python
assert actual_ids == expected_ids
```

件数の一致では取り違えを検出できず、集合の一致では順序の誤りを検出できない。今回はAPIの契約に降順が含まれるため、列をそのまま比較した。[バックフィル後の集合と重複の確認](./102-backfill-verify-identifier-sets.md)とは、比較する対象と確認したい契約が異なる。

## つまずいたところ

この実DB照合について、記録された実行結果に失敗はなかった。Docker内でpytestを実行する前提を揃えた[記事137](./137-run-pytest-in-built-container.md)とは異なり、今回は起動中のHTTP経路と既存MySQLを結び付けて確認した。

## 学んだことと制約

**FACT:** 4モードすべてで、代表1著者のAPIのID列とSQLのID列が順序まで一致した。これは実際の設定、稼働API、MySQLを通した確認である。

**INFERENCE:** ヘルスチェック、公開スキーマ、ID列の比較を分けると、起動、契約の反映、読み取り結果をそれぞれの証拠で判断しやすい。

全著者・全日付を実DBで確認したわけではない。正確な日本時間の境界、100件超の返却、未知の著者等の契約は、人工データを使う24ケースのHTTPテストが補っている。これらのテストはSQLiteを使うため、実MySQLによる代表例の確認とも役割が異なる。

大きな範囲の性能測定、更新中データの一貫性、リモート環境へのデプロイは、この結果からは保証できない。

## まとめ

DBを使うAPIの更新確認では、起動したことと、目的の結果を返したことを分けて記録する。今回の日付条件では、4モードそれぞれについて、著者条件と明示したUTC境界を持つSQLのID列を期待値とし、稼働APIの応答を順序まで照合した。

## 根拠

- slope-collectorの`compose.yaml`、`app/api/records.py`、`tests/medium/test_record_date_range.py`。
- 作業時の実行結果の保管要約：`output/entity-date-publication/runtime-verification.md`。再実行ログではなく、実行済み操作と観測結果の要約。
- 作業時のコード検証記録：`output/reviews/integrated-entity-date-range.md`。
- [今回のAPI変更のPR #43](https://github.com/kishimin/slope-collector/pull/43)。ローカルDockerでの確認と、PRの公開・マージは別の結果である。
