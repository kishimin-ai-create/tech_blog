# 404画面と予期しないエラー画面を振る舞いから実装する

## 結論

404画面とErrorFallbackを、表示文言・復旧操作・アクセシビリティというユーザー観測可能な振る舞いごとに小さくテストし、Red→Greenを繰り返すことで、画面の責務を保ったまま実装できます。

## 対象の分離

404画面はルーティングされた存在しないパスの復旧を担当し、ホームへのリンクを提供します。ErrorFallbackはルートErrorBoundaryが捕捉した予期しない例外からの復旧を担当し、Header/Footerを表示せず、ページ再読み込みを提供します。両者を同じテストへ詰め込まず、それぞれの画面の公開された振る舞いを検証しました。

## TDDの進め方

- 404画面: 日本語表示、英語表示、ホームリンク、見出しとリンクのアクセシブル構造
- ErrorFallback: 日本語・英語表示、保存ロケール優先、ブラウザ言語フォールバック、既定ロケール、再読み込み

各ケースで先に失敗するテストを追加し、最小実装で成功させました。テストではCSSクラスや内部コールバックではなく、ロール、表示文言、リンク先、アクセシブルな名前を確認しています。

## 検証結果

NotFoundViewのSmallテストは4件、ErrorFallbackは5件が成功しました。全Smallテストは126件成功し、1件はブラウザの`location.reload`を再定義できないためtodoです。型チェック、Lint、対象ファイルのPrettier確認も成功しました。

## 事実・判断・未確認事項

- FACT: 404画面とErrorFallbackの実装・Smallテストを別コミット系列で追加した。
- FACT: ErrorFallbackは再読み込み実動作のテストを保留している。
- INFERENCE: 画面ごとの復旧責務を分離すると、ルーティング境界とErrorBoundary境界のテスト重複を避けられる。
- ASSUMPTION: 実ブラウザでの視覚表示とリロード後の初期表示は未確認である。

## 参考

- `docs/v1/ui/ui.md §19–20`
- `frontend/src/features/not-found/views/NotFoundView.small.test.tsx`
- `frontend/src/features/error/views/ErrorFallback.small.test.tsx`
- `331e418 test: specify Japanese not-found rendering`
- `50db612 test: specify Japanese error fallback content`
