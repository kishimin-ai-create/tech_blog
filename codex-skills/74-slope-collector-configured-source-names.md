---
title: "収集処理の内部キーとDB表示名を分離してチェックポイントを守る"
tags: [Python, MySQL, 設計, スクレイピング]
---

# 収集処理の内部キーとDB表示名を分離してチェックポイントを守る

## 結論

収集処理が使う`source_a`のような内部キーと、DBやAPIに表示するsource名は同じ値として扱わない方がよい。内部キーは設定の切り替えに使い、表示名は`.env`から注入してRepositoryへ渡すと、表示名を変更しても既存の収集チェックポイントを再利用できる。

## 背景

sourceのDB行を表示名へ変更したところ、既存コードは`Source.name == record.source_key`で検索していた。内部キーが`source_a`、DBの表示名が別の値になると検索条件が一致しない。

この不一致を放置すると、次回収集で既存行を見つけられず、新しいsource行を作る可能性がある。収集済みレコードの重複判定もsource行を起点にしているため、単なる表示上の変更では済まない。

## 分離する値

| 値 | 役割 | 例 |
| --- | --- | --- |
| 内部キー | 設定プレフィックス、ログ、処理分岐 | `source_a` |
| 表示名 | DBの`Source.name`、ローカルAPI表示 | `.env`で設定 |

設定モデルに表示名を追加し、収集レコードへ保持させる。

```python
return CollectedRecord(
    source_key=self._source_key,
    source_name=self._config.name,
    # other fields...
)
```

Repositoryは表示名を検索に使い、未設定時だけ内部キーへ戻す。

```python
source_name = record.source_name or record.source_key
source = session.scalar(
    select(Source).where(Source.name == source_name)
)
```

## 検証

既存の表示名を持つsource行と同じ`source_name`を渡し、Repositoryが新しいsource行を作らず既存レコードを検出するテストを追加した。全体では77テストが成功し、カバレッジは86.21%だった。

## 注意点

この設計だけでは、表示名の一意性や変更履歴までは管理しない。表示名を利用者が変更できる機能を追加する場合は、内部キーをDB列として永続化する設計や、表示名変更の監査方針を別途決める必要がある。

## まとめ

外部取得処理の安定した内部キーと、人間が読むDB表示名は責務が異なる。チェックポイントや重複判定が表示名に依存する場合でも、両者の受け渡しを設定境界で明示すれば、名前変更による既存データの切断を防ぎやすい。

## 参考

- `app/config.py`
- `app/scraping/domain.py`
- `app/repositories/collection.py`
- `tests/medium/test_collection_repository.py`
