# 一覧APIで「未登録の取得元」と「空の一覧」を区別する

## 結論

`GET /entities?source_id=...`では、取得元が存在しない場合を404、存在する取得元にデータがない場合を200の空配列にした。入力値の形式検証と、DB上の存在確認を分けることで、呼び出し側が選択ミスと正常な空状態を区別できる。

これはslope-collectorで合意した契約であり、すべての検索APIに404を要求する一般則ではない。

## 背景と課題

修正前は、`source_id`を一覧の絞り込み条件にだけ使っていた。登録されていない正の整数を指定しても、該当行がないため200と`{"entities":[]}`が返っていた。一方、0以下は既存の`Query(ge=1)`で422になっていた。

今回必要だったのは、数値の上限を現在の最大IDで固定することではなく、指定された取得元がDBに存在するか確認することだった。IDには欠番があり、新しい取得元も追加される。

## 前提・制約

- Python 3.14.7、FastAPI、SQLAlchemyを使用する開発用の読み取りAPI。
- 既存の`NOT_FOUND`と`VALIDATION_ERROR`の形式を再利用する。
- 取得元が存在しても所属データが0件という状態は正常として扱う。
- フィルターを省略した一覧取得は従来どおり返す。

## 検討した選択肢

| 観点 | 現状の絞り込みだけ | 取得元の存在確認を追加 |
| --- | --- | --- |
| 未登録と空状態の区別 | どちらも200の空配列 | 404と200で区別 |
| DBアクセス | 一覧取得 | 指定時は存在確認も必要 |
| 既存クライアントへの影響 | 変更なし | 未登録IDの扱いが変わる |

利用者の要求に合わせて存在確認を採用した。存在確認のための追加照会が発生する点はトレードオフである。性能改善の計測は行っていない。

## 実装

ルーター関数の文数がLintの上限を超えたため、存在確認と取得を小さな関数へまとめた。

```python
def _entities_for_source(session: Session, source_id: int | None) -> list[Entity]:
    if source_id is not None and session.get(Source, source_id) is None:
        raise _not_found()
    statement = select(Entity).order_by(Entity.id)
    if source_id is not None:
        statement = statement.where(Entity.source_id == source_id)
    return list(session.scalars(statement).all())
```

既存の例外ハンドラーにより、未登録の場合は次の応答になる。

```json
{"code":"NOT_FOUND","message":"Requested resource does not exist."}
```

## 検証

取得元IDを1と3にして、間の2を欠番にしたテストを用意した。単なる最大ID判定で通ってしまう実装を避けるためである。SQLiteとASGI経由の回帰テストはMediumに配置した。

| 入力・状態 | 期待結果 |
| --- | --- |
| 未登録の2、999999 | 404 |
| 登録済みで所属データなしの3 | 200・空配列 |
| データありの1 | 200・該当データ |
| フィルター省略 | 200・全データ |
| 0、-1、整数以外 | 422 |

修正前は404を期待する2件が失敗し、残り6件が成功した。修正後は8件すべて成功。全体では`uv run pytest acceptance tests --cov=app --cov-report=term-missing --cov-report=xml -q`が196件成功し、行91.28%、分岐84.23%、総合89.91%だった。Ruffの整形・Lint、mypyも成功した。

Docker再構築後も、未登録IDの404、0の422、登録済みIDの200を実際のHTTP応答で確認した。

## 影響と残る確認

未登録IDを正常な空状態として扱っていたクライアントは、404を扱う必要がある。登録済みの空状態は回帰テストで確認し、実DBにテスト用取得元を追加しての確認はしていない。GitHub上のCI結果やマージ結果は、これらのローカル検証とは別に確認する。

## 根拠

- [回帰テストのコミット efb79c7](https://github.com/kishimin/slope-collector/commit/efb79c770bc47cc2db66e22c12ffb8633bbd392f)
- [実装とAPI仕様のコミット 4aa3867](https://github.com/kishimin/slope-collector/commit/4aa3867e82afab713c06891732865e456cccc09c)
- 対象：`app/api/records.py`、`tests/medium/test_entity_source_filter.py`、`docs/api/README.md`
