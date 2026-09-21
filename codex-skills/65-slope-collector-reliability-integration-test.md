---
title: "収集境界のリトライと失敗通知をモックHTTPで統合テストする"
tags: [Python, スクレイピング, テスト, 運用]
---

# 収集境界のリトライと失敗通知をモックHTTPで統合テストする

## 対象読者

HTTP取得、保存、失敗通知が分かれた収集処理を、外部サイトへ接続せずに検証したい開発者を対象にする。

## 結論

リトライと失敗通知は、それぞれの関数を単体で呼ぶだけでなく、モックHTTPから収集結果と通知までをつないだ統合テストで確認する。今回のテストでは、一時的な503を指数バックオフ後に再試行し、解消しない取得失敗は本文を含めない1通のサマリーに変換する契約を固定した。

## 一時的な失敗を再試行する

`httpx.MockTransport`で一覧取得の最初の2回だけ503を返し、3回目に匿名化したHTMLを返す。待機関数を差し替えることで実時間を消費せず、呼び出し回数と待機値を観測できる。

```python
assert result.saved_records == 1
assert result.failed_records == 0
assert list_attempts == 4
assert [delay for delay in delays if delay > 0] == [2, 4]
```

テストが確認するのは、再試行したことだけではない。最終的に保存まで到達したこと、失敗件数が残っていないこと、待機時間が設定値から計算されていることも同時に確認する。

## 解消しない失敗を1通にまとめる

詳細取得を400で失敗させ、`notify_failures`へ収集結果を渡す。SMTPはテスト用オブジェクトへ差し替え、外部メールサービスへ接続しない。

```python
assert result.failed_records == 1
assert len(sent) == 1
assert "stage=record" in content
assert "exception=FetchPermanentError" in content
assert "response body" not in content
```

この境界では、運用者が原因の分類を読める一方、取得した本文や完全なレスポンスを通知へ含めないことを確認している。実際の取得先や本文はテストデータに持ち込まない。

## 検証結果と制約

`tests/medium/test_collection_reliability_integration.py`の2テストが成功し、リポジトリ全体では73件、カバレッジ87.12%を確認した。Ruff、mypy、`git diff --check`も成功している。

これはモックHTTPとテスト用SMTPによる契約確認であり、実際の外部サイトの応答やメール配送を証明するものではない。実環境の取得は実行していない。

## まとめ

通信の再試行と通知の安全性は、HTTPクライアント単体のテストだけでは確認しきれない。匿名化したモックと差し替え可能な待機・SMTP境界を使い、取得から保存、失敗サマリーまでを1本のシナリオで検証すると、運用上の契約を保ったまま外部負荷を発生させずに確認できる。

## 根拠

- `tests/medium/test_collection_reliability_integration.py`
- `app/services/collection.py`
- `app/services/notifications.py`
- `1bbed02 test: cover collector reliability integration`
- `ba1e29c docs: record reliability verification`

確認日: 2026年9月21日
