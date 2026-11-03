# Qiita CLIで投稿済み記事のIDがないときの同期

## 結論

Qiita CLIで投稿済みの原稿に `id: null` が残ると、`publish --all` が新規投稿として扱う場合がある。既存原稿を一時的に `ignorePublish: true` にし、投稿済み記事のIDが入ったファイルと区別することで、対象を整理できる。

## 発生した問題

GitHub Actionsの `qiita publish --all` が `title has already been used` で失敗した。原稿には同名の投稿済み記事がある一方、`id: null` が残っていた。エラーログだけでは、失敗前に他の記事が処理されたかは確定できない。

`qiita pull` では、投稿済み記事のIDを名前にしたファイルが `public/` に作られた。これは、既存原稿が正しいIDを保持していることの代わりにはならない。同じタイトルで別IDの記事もあるため、タイトルだけで対応を確定しない。

## 対応

過去のID未設定原稿を `ignorePublish: true` とし、投稿済みの別記事には実際のIDを反映した。新規記事をpushすると、Actionsは投稿に成功し、IDと更新日時を書き戻した。

ただし公式Actionは `public/*` をステージするため、この実行では取得したID名ファイル13件もコミットされた。ローカルに同名の未追跡ファイルがあり、fast-forwardが拒否された。

## 動作確認

13件をGitHub側とバイト単位で比較し、一致を確認して退避した後、fast-forwardした。その後、投稿対象外の下書きをpushした [Actions実行](https://github.com/kishimin/qiita-tech-blog/actions/runs/41587055526) は成功し、ローカルとリモートのSHAも一致した。別の[新規記事投稿](https://github.com/kishimin/qiita-tech-blog/actions/runs/41585102063)はタイトル重複で失敗し、[修正後の投稿](https://github.com/kishimin/qiita-tech-blog/actions/runs/41586730542)は成功した。

## 残る注意点

- 投稿済み記事を編集する際は、IDと本文を照合して正本を選ぶ。旧ファイルの除外は重複記事の削除ではない。
- 未完成記事は `ignorePublish: true` にする。`private` は公開範囲の指定で、投稿をスキップする設定ではない。
- Actionsが書き戻したコミットを、次の作業前に取り込む。未追跡ファイルとの衝突時は内容を保全してから比較する。
- Qiita側の重複記事の整理や、今後のすべての投稿成功は確認していない。

## 参考

- [Qiita CLI公式README](https://github.com/increments/qiita-cli)
- [公式publish Action](https://github.com/increments/qiita-cli/blob/main/actions/publish/action.yml)

確認日: 2026-11-03
