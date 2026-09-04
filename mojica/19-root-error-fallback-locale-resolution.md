# ルートErrorFallbackをi18n Providerから独立させる設計

## 結論

ルートのErrorFallbackは通常のi18n Contextへ依存せず、最小限の辞書とロケール解決処理を自身に持たせると、Provider障害時にも翻訳済みの復旧画面を表示できます。

## 背景

ErrorBoundaryをProviderの外側に置くと、I18nProvider自身が失敗した場合でもフォールバック画面を描画できます。その一方で、ErrorFallbackがI18nProviderへ依存すると、障害境界の内側に依存が戻ってしまいます。

## 解決規則

今回の実装では、次の順序で表示ロケールを決定します。

1. `localStorage`の対応ロケール
2. `navigator.languages`の先頭から最初に対応する言語
3. 既定値の`ja`

保存値の読み取りで例外が起きても、ブラウザ言語と既定値による復旧を継続します。対応ロケールは辞書のキーから導出し、言語追加時に固定条件分岐を増やさない構造にします。

## 検証

ErrorFallbackのSmallテストで、日本語・英語の表示、保存ロケールの優先、ブラウザ言語へのフォールバック、未対応または読み取り不能時の日本語フォールバックを確認しました。対象テストは5件が成功し、再読み込みの実動作だけはVitest Browser環境で`location.reload`を再定義できないため未実装です。

## 事実・判断・未確認事項

- FACT: `ErrorFallback`はHeader/Footerを描画せず、独立した辞書から文言を選択する実装になった。
- FACT: ロケール解決に関するSmallテストは成功した。
- INFERENCE: Provider障害時の表示継続と、対応言語追加時の分岐増加抑制を両立できる。
- ASSUMPTION: 実ブラウザでのリロード動作は別環境で確認が必要である。

## 参考

- `docs/v1/ui/ui.md §20`
- `docs/v1/ui/components/ErrorFallback.md`
- `C:/Users/Kazum/.codex/docs/adr/0049-keep-root-error-fallback-locale-extensible.md`
- `754ec61 feat: prefer persisted error fallback locale`
