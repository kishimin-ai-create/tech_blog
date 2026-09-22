---
title: "一覧ラッパー内の全リンクを抽出し、記事の取りこぼしを防ぐ"
tags: [Python, スクレイピング, テスト, データ収集]
---

# 一覧ラッパー内の全リンクを抽出し、記事の取りこぼしを防ぐ

## 結論

HTMLの一覧要素が「1記事」ではなく「ページ内の記事群を包むラッパー」だった場合、`select_one()`は最初の記事だけを返します。記事リンクを全件抽出し、記事キーを実行中とDBの両方で重複排除することで、ページ内の後続記事を取りこぼさずに処理できます。

今回の`slope-collector`では、対象メンバーのバックフィルを実行した結果、従来の1ページ1記事相当の取得から、1ページ内の全記事処理へ修正できました。

## 発生した問題

メンバーを絞り込んだ一覧ページをページ番号順に取得していましたが、ログ上では1ページにつき1件の詳細ページしか開かれていませんでした。一方、実HTMLを確認すると、一覧セレクタに一致する要素の中に複数の記事詳細リンクが存在しました。

問題の実装は次の形です。

```python
for item in soup.select(list_item_selector):
    link = item.select_one(detail_link_selector)
```

`list_item_selector`が記事カードごとの要素なら成立します。しかし、ページ内の記事をまとめる`ul`のような要素に一致すると、`select_one()`によって最初のリンクだけが選ばれます。

## 原因

一覧要素の粒度を「1要素=1記事」と仮定していたことが原因でした。セレクタ名だけでは要素の責務を保証できないため、実HTMLで次の件数を確認する必要があります。

```text
一覧ラッパーの件数
詳細リンクの件数
ページネーションリンクの件数
```

一覧ラッパーが1件でも、その配下の詳細リンクが複数件なら、記事の繰り返し単位はラッパーではなくリンク群です。

## 解決方法

各一覧要素から詳細リンクを1件選ぶのではなく、該当リンクをすべて走査します。

```python
for item in soup.select(list_item_selector):
    links = item.select(detail_link_selector)
    if not links:
        raise ParseContractError("required element is missing")

    for link in links:
        source_path = normalize_source_path(link["href"])
        external_key = extract_record_id(source_path)
        if external_key not in seen:
            seen.add(external_key)
            records.append(RecordReference(external_key, source_path))
```

さらに、対象メンバーの一覧ページでは、ページ内の記事キーをまとめてDBへ照会します。

```text
ページ内の記事キーを抽出
  ↓
DBへIN検索を1回実行
  ↓
既存キーを除外
  ↓
未取得記事だけ詳細HTTP
```

同じ記事キーが複数ページに現れた場合も、実行中の集合で重複を除外し、詳細HTTPを2回開かないようにしました。

## テストで固定した振る舞い

一覧ラッパー1つの中に2つの詳細リンクがある入力を追加し、2件とも抽出されることを確認しました。また、ページ巡回テストの期待値を、同一記事を複数回保存しない契約に合わせました。

## 動作確認

修正後に、次の検証を実行しました。

```powershell
python -m ruff format --check .
python -m ruff check .
python -m mypy app
python -m pytest acceptance tests --cov=app --cov-report=term-missing -q
```

結果は、76テスト成功、カバレッジ86.17%でした。

さらに、ローカルDBへ対象メンバーのバックフィルを実行し、一覧ページ内の複数記事を処理できることを確認しました。

| 対象 | ページ | 保存 | 既存スキップ | 失敗 | DB合計 |
| --- | ---: | ---: | ---: | ---: | ---: |
| source_bの対象entity 1 | 8 | 70 | 18 | 0 | 88 |
| source_bの対象entity 2 | 5 | 37 | 13 | 0 | 50 |
| source_bの対象entity 3 | 21 | 226 | 20 | 0 | 246 |

source_aでも既存記事を詳細HTTP前にスキップできることを確認しました。ただし、2つの対象では詳細解析の失敗が残っており、全件成功とは扱っていません。

## 事実・推論・未確認事項

- FACT: 一覧ラッパー1件の配下に複数の詳細リンクが存在した。
- FACT: `select_one()`から全リンク走査へ変更した。
- FACT: 修正後のテストは76件成功した。
- FACT: source_bの3対象では失敗0件でDB件数を照合できた。
- INFERENCE: 一覧セレクタは、見た目のカードではなく実際のDOM上の繰り返し単位として検証する必要がある。
- 未確認: source_aで詳細解析に失敗した記事のHTML差分は、今回の変更では特定していない。
- 未実装: sourceごとの専用アダプター分離と、entity単位の日付増分を使う定期取得の再設計は別課題である。

## まとめ

スクレイピングの一覧解析では、`select_one()`が常に誤りなのではなく、セレクタが指す要素の粒度と組み合わせる必要があります。実HTMLのリンク件数を確認し、全件抽出、ページ単位のDB重複確認、実行中の重複排除を組み合わせることで、取得件数と保存件数を検証可能にできます。

## 参考

- `app/scraping/adapter.py`
- `app/repositories/collection.py`
- `app/services/collection.py`
- `tests/small/test_source_adapter.py`
- `tests/medium/test_collection_sequence.py`
- `slope-collector` commit `ae2de22`

確認日: 2026年9月22日
