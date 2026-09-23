---
title: "一覧APIへ本文を追加するときに詳細レスポンスとの重複を防ぐ"
tags: [Python, FastAPI, API, テスト]
---

# 一覧APIへ本文を追加するときに詳細レスポンスとの重複を防ぐ

## 背景

ローカル確認APIの`GET /records`は、レコードIDやタイトルなどのメタデータだけを返していた。保存済み本文を確認するには、各レコードの詳細APIを個別に呼ぶ必要があった。

## 変更方針

一覧レスポンスの共通モデルへ`body`を追加し、詳細レスポンスも同じ変換結果を継承する構造にした。

```python
class RecordListItem(BaseModel):
    id: int
    entity_id: int
    title: str
    body: str
    source_url: str
```

変換処理でもDBの本文を設定する。

```python
return RecordListItem(
    id=record.id,
    entity_id=record.entity_id,
    title=record.title,
    body=record.body,
    source_url=record.source_url,
    published_at=_as_utc(record.published_at),
)
```

## 実装中に起きた失敗

詳細モデルは一覧モデルを継承していたため、一覧変換へ`body`を追加したあとも、詳細レスポンス生成で`body=record.body`を明示していた。結果として同じキーワード引数を2回渡す`TypeError`になった。

詳細側の重複指定を取り除き、共通変換結果へ本文を一元化した。

## テスト

一覧APIの成功レスポンスへ本文を追加し、詳細APIにも本文が残ることを受け入れテストで確認した。実行結果は全77テスト成功、カバレッジ86.21%だった。

## 制約

本文を一覧へ含めると、1回のレスポンスサイズは増える。今回のAPIはローカル確認用途であり、ページングは既存のままなので、公開APIや大量本文を扱う用途へ同じ契約を適用する場合は、レスポンスサイズと取得単位を再評価する必要がある。

## まとめ

一覧と詳細で共有モデルを使う場合、共通モデルにフィールドを追加したら、継承先での明示指定を確認する。レスポンス契約を先にテストへ追加すると、一覧の不足と詳細の二重指定を別々の失敗として検出できる。
