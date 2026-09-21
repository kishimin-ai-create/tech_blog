---
title: "増分収集を保存済みチェックポイントで停止し、古い履歴の再取得を抑える"
tags: [Python, スクレイピング, バッチ処理, テスト]
---

# 増分収集を保存済みチェックポイントで停止し、古い履歴の再取得を抑える

## 対象読者

新しい順にページを取得する収集処理で、定期実行時の再取得量を減らしたい開発者を対象にする。

## 結論

増分収集では、保存済みレコードに到達した時点で、その実行の古いページ探索を止める。保存処理の重複防止だけに任せず、次のHTTPリクエストを発生させない停止条件としてチェックポイントを扱う。

今回のIssue #8では、同じ入力を2回実行したとき、2回目は保存済みレコードの詳細を1件確認したところで終了し、それより古いページを取得しないことをモックHTTPで固定した。

## なぜ重複防止だけでは足りないのか

DBの一意制約があれば、同じレコードを2回保存することは防げる。しかし、保存済みかどうかを確認するために古い一覧ページや詳細ページへアクセスし続けると、DB上の重複はなくてもHTTP通信は重複する。

日次処理では、最新の記事から順に取得し、最初に保存済みの識別子へ到達したら、その先は前回より古い履歴だと判断できる。この停止条件は、保存時の一意制約とは異なる責務である。

## テストで固定した契約

テストでは、外部サイトへ接続せず、匿名化したHTMLを`httpx.MockTransport`で返した。最初の実行でレコードを保存し、2回目の実行で同じ一覧と詳細を返す。

```python
first = collect_all(
    settings=settings,
    source_keys=("source_a",),
    mode=CollectionMode.DAILY,
    repository=repository,
    transport=mock_transport,
    sleep=lambda _delay: None,
)
second = collect_all(
    settings=settings,
    source_keys=("source_a",),
    mode=CollectionMode.DAILY,
    repository=repository,
    transport=mock_transport,
    sleep=lambda _delay: None,
)

assert first.saved_records == 1
assert second.saved_records == 0
assert second.skipped_records == 1
assert requested_paths == ["/list", "/detail/1"]
```

実際のテストでは、固有の取得先を表す値を使わず、保存済みキーを保持するテスト用リポジトリと匿名化したパスを使用した。

## バックフィルとの境界

この停止条件は日次の増分モードに適用する。初回バックフィルは過去の履歴を探索する目的が異なるため、同じ停止条件をそのまま適用すると未取得の古いレコードを取りこぼす可能性がある。

したがって、コマンドとモードを分け、定期処理は増分収集、初回処理は手動バックフィルとして扱う。長時間バックフィルの分割や永続カーソルは別の設計課題であり、本記事では実装済みとはしない。

## 検証結果と制約

Issue #8のブランチでは、増分停止のmediumテストが1件成功し、リポジトリ全体では71件、カバレッジ86.20%を確認した。Ruff format、Ruff lint、mypy、`git diff --check`も成功した。

このテストはモックHTTPとテスト用リポジトリによる契約確認であり、実環境のページ順序が常に新しい順であることや、実際の全履歴が完全に取得できることは証明しない。

## まとめ

増分収集のチェックポイントは、重複保存を防ぐDB制約と、古いページを読まないHTTP停止条件の両方で利用する。日次処理と初回バックフィルを別モードに保つことで、取りこぼしを避けながら定期実行の通信量を抑えられる。

## 根拠

- `tests/medium/test_collection_incremental.py`
- `app/services/collection.py`
- `fc15ff6 test: cover incremental checkpoint stopping`
- `fa4682b docs: record MVP verification results`

確認日: 2026年9月21日
