---
title: "収集レコードと関連アセットを1トランザクションで保存する"
tags: [Python, SQLAlchemy, データベース, テスト]
---

# 収集レコードと関連アセットを1トランザクションで保存する

## 対象読者

1つの収集結果に本文レコードと関連アセットが含まれるアプリケーションで、途中保存による不整合を防ぎたい開発者を対象にする。

## 結論

収集したレコードと関連アセットは、1回の永続化処理・1トランザクションで保存する。アセットの制約違反など後半の保存に失敗した場合は、先に作ったソース、Entity、レコードも残さない。

## 起きやすい不整合

保存処理を段階的にcommitすると、本文レコードだけが残り、アセット保存だけ失敗する状態が起こり得る。次回収集では本文が既存と判定され、欠落したアセットを補えない可能性もある。

さらに、表示名は変更される可能性があるため、重複判定には表示名ではなく、ソースと対象Entityの安定した外部識別子を使う必要がある。

## 実装で確認した契約

SQLAlchemyのテスト用SQLiteデータベースへ、1件のレコードと1件のアセットを保存する。続けて同じ入力を保存すると、ソース、Entity、レコード、アセットが1つずつだけ残る。

別のテストでは、同じ表示名でも外部識別子が異なる2つのEntityを別物として保存し、同じ外部識別子で表示名だけ変わった入力は既存Entityを更新することを確認した。

```python
assert repository.persist(collected_record()) is True
assert repository.persist(collected_record()) is False

with sessions() as session:
    assert len(session.scalars(select(Source)).all()) == 1
    assert len(session.scalars(select(Entity)).all()) == 1
    assert len(session.scalars(select(Record)).all()) == 1
    assert len(session.scalars(select(Asset)).all()) == 1
```

## ロールバックをテストする

同じレコードに同じ位置のアセットを2つ指定し、アセットの一意制約違反を発生させる。`IntegrityError`が送出された後、4種類のテーブルを確認し、すべて空であることを検証する。

```python
with pytest.raises(IntegrityError):
    repository.persist(invalid_record)

with sessions() as session:
    assert session.scalars(select(Source)).all() == []
    assert session.scalars(select(Entity)).all() == []
    assert session.scalars(select(Record)).all() == []
    assert session.scalars(select(Asset)).all() == []
```

このテストが確認するのは、例外が発生した事実だけではない。失敗後に部分的なチェックポイントが残っていないことまで、永続化の契約として固定している。

## 検証結果と制約

Issue #8のmediumテストでは、重複保存、安定識別子、既存データの引き継ぎ、表示名変更、トランザクションロールバックを確認した。ブランチ全体では71件のテストが成功し、カバレッジは86.20%だった。Ruff format、Ruff lint、mypy、`git diff --check`も成功した。

テストはSQLiteで実行しているため、MySQL固有のDDLや隔離レベルまで検証した結果ではない。空のMySQLデータベースに対するAlembic検証はIssue #8の未完了項目として残っている。

## まとめ

収集結果を複数テーブルへ保存する場合、重複防止とトランザクション境界を別々に考える。安定した外部識別子で既存データを認識し、関連データを同じトランザクションで保存すれば、後半の制約違反で部分的なチェックポイントが残ることを防げる。

## 根拠

- `tests/medium/test_collection_repository.py`
- `app/repositories/collection.py`
- `392873f test: verify collection transaction rollback`
- `72885e6 docs: record rollback verification`

確認日: 2026年9月21日
