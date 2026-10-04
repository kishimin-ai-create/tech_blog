# Dockerのenv-fileでURLに引用符が残った原因と対応

## 結論

Dockerのenv-fileで注入したcollectorのURLに二重引用符が残り、RequestsのInvalidSchemaが発生した。利用者は値を囲む引用符を削除し、このエラーが解消したと報告した。

## 発生した問題

2026年10月4日、Docker Desktop上の画像APIを次の形式で起動した。

```powershell
docker run --env-file .env -p 8000:80 blog-wordcloud-analysis:local
```

画像APIのリクエストがHTTP 500になった。ファイルの問題が疑われたが、ログではcollectorのsourcesを取得する段階で失敗していた。

```text
InvalidSchema: No connection adapters were found for '"http://127.0.0.1:8080"/sources'
```

## 調査

起動自体とhealth・docsは成功していた。スタックトレースを確認すると、生成ファイルの読み込みではなくrequests.getで失敗していた。ネットワークの到達性を調べる前に、URL文字列に引用符が含まれることを確認できた。

## 原因

設定コードがos.environから取得したベースURLへsourcesのパスを追加している。環境変数に残った引用符も含めてURLを組み立てたため、RequestsがHTTP URLとして扱えなかった。

この観測はDockerのenv-fileで渡した値についての結果である。Pythonのdotenvによるファイル読み込みと同じパース結果になるとは限らない。

## 解決方法

値を囲む引用符を削除した。説明用の例:

```dotenv
# 変更前
SLOPE_COLLECTOR_URL="http://collector.example:8080"
# 変更後
SLOPE_COLLECTOR_URL=http://collector.example:8080
```

このURLは実際の接続先を掲載したものではない。利用者が引用符削除でエラーが解消したと報告した。さらにネットワークの接続拒否が現れたため、引用符修正だけで画像生成全体が成功したとは記録しない。

## 動作確認

| 確認事項 | 結果 |
| --- | --- |
| 修正前の原因 | ログの引用符付きURLとInvalidSchemaを直接確認 |
| 引用符削除による改善 | 利用者の報告 |
| その後のエラー | 別の接続拒否をログで確認 |

## 再発防止とまとめ

PROPOSAL: 環境変数の入力経路と、アプリが受け取る値を区別する。ログの最初の失敗箇所から原因を調べ、URL形式の失敗と接続先の失敗を一緒に扱わない。恒久的な入力検証は追加していない。

## 参考

- [Docker runの環境変数とenv-file](https://docs.docker.com/reference/cli/docker/container/run/#env)
- [次に発生した接続先の問題](./95-container-loopback-and-local-collector.md)
