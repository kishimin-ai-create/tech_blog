# 管理画面の並べ替えと削除の完了を伝える

## 結論

管理画面の一覧では、並べ替えや削除の成功時に対象を既存の status 領域へ表示するようにした。次の操作が始まると古い成功表示を消し、失敗時には既存のエラーを示す。各変更はドラフト PR であり、アプリの main には未マージである。

## 背景

操作後の一覧順序や項目数の変化は画面上で分かるが、その変化を視覚に頼らず確認する手段が不足していた。結果を既存の状態通知に表示し、失敗後に以前の成功が残らないことも検証した。

## 対応

作品・スキル・お気に入りの並べ替えは [#290](https://github.com/kishimin-ai-create/profile-hub/pull/290)、[#291](https://github.com/kishimin-ai-create/profile-hub/pull/291)、[#292](https://github.com/kishimin-ai-create/profile-hub/pull/292)、リンクは [#293](https://github.com/kishimin-ai-create/profile-hub/pull/293) で対応した。各通知は操作対象の名前または URL を含む。

削除完了の通知は [#296](https://github.com/kishimin-ai-create/profile-hub/pull/296)、[#297](https://github.com/kishimin-ai-create/profile-hub/pull/297)、[#298](https://github.com/kishimin-ai-create/profile-hub/pull/298) で作品・スキル・お気に入りに追加した。削除を取り消した場合は新たな成功通知を出さず、要求が失敗した場合は古い成功表示を消す。

## 検証と限界

各対象テストで成功後に次の操作を失敗させ、古い成功表示が空になることを確認した。並べ替え完了の表示は [#294](https://github.com/kishimin-ai-create/profile-hub/pull/294) と [#295](https://github.com/kishimin-ai-create/profile-hub/pull/295) で Chromium と WebKit でも確認した。ブラウザーテストの API 応答は制御されている。

DOM の status と実ブラウザーでの表示を確認したが、実際のスクリーンリーダーによる読み上げは未確認である。状態通知の確認だけでアクセシビリティ全体の適合は断定できない。
