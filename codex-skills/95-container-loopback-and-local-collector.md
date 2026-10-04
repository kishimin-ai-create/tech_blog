# コンテナの127.0.0.1でcollectorへ接続できなかった原因

## 結論

コンテナ内の127.0.0.1は、そのコンテナ自身を指す。Windows側へ公開されたcollectorの8080番へ向かう設定にはならない。Docker版からhost.docker.internal経由のhealth到達性を確認し、ローカルのuv起動では127.0.0.1へ接続先を上書きして画像APIの成功を確認した。

## 発生した問題

URLの引用符を直した後、sources取得が次のエラーになった。

```text
HTTPConnectionPool(host='127.0.0.1', port=8080):
Max retries exceeded with url: /sources
Caused by NewConnectionError: [Errno 111] Connection refused
```

NewConnectionErrorは新規接続を確立できなかったことを表す。Max retries exceededの表示だけから、実際に大量のリトライをしたとは断定しない。

## 調査と原因

2026年10月4日の確認結果:

| サービス | ホスト側の公開ポート | コンテナ内のポート |
| --- | --- | --- |
| 画像API | 8000 | 80 |
| collector | 127.0.0.1:8080 | 8080 |

ホストのブラウザから127.0.0.1:8080へアクセスする場合と、画像APIコンテナ内のPythonから同じ文字列へ接続する場合では、指す環境が違う。ポート番号で127.0.0.1の指す環境が切り替わるわけではない。

-p 8000:80はホストから画像APIへ入る転送であり、画像APIからcollectorへの送信先を変更しない。[Dockerのポート公開](https://docs.docker.com/get-started/docker-concepts/running-containers/publishing-ports/)

## 解決の確認

Docker Desktop公式ではhost.docker.internalはホストの内部IP、gateway.docker.internalはDocker VMのゲートウェイを指す。今回はホストの公開ポートへ接続するため前者を使う。[公式の名前の対応表](https://docs.docker.com/desktop/features/networking/networking-how-tos/#connect-a-container-to-a-service-on-the-host)

```text
画像APIコンテナ
  → host.docker.internal:8080
  → ホストの公開ポートから転送
  → collectorコンテナ:8080
```

FACT: この経路でcollectorのhealthはHTTP 200、status: okを返した。gateway経由は未検証であり、Docker版の画像生成リクエストはこの変更後に確認していない。

## ローカルのuv起動では別の接続先を使う

同じ.envのDocker用URLでWindows上のuvプロセスを起動すると、画像API自身のhealthは成功したが、collectorはNameResolutionErrorになった。このWindows環境ではhost.docker.internalを名前解決できなかった。

.envを変更せず、起動プロセスの環境変数だけを上書きした。services/analysisで実行したコマンド:

```powershell
$env:SLOPE_COLLECTOR_URL = 'http://127.0.0.1:8080'
uv run --locked uvicorn main:app --host 127.0.0.1 --port 8001
```

設定コードはload_dotenvの既定動作で既存環境変数を上書きしない。このプロセスにはローカル用URLが適用された。

## 動作確認

| 確認 | 実測結果 |
| --- | --- |
| ローカルuvのバージョン | 0.12.15 |
| ローカルAPIのhealth・docs・openapi.json | HTTP 200 |
| 設定を読むuvプロセスからcollectorのhealth・sources | HTTP 200 |
| ローカル画像API | 1対象でHTTP 200、224,252バイトの応答 |
| Docker版からcollectorのhealth | HTTP 200 |

全対象、アップロード画像、ブラウザでの描画は検証していない。個人の識別につながる対象別IDと取得本文は掲載しない。

## 再発防止とまとめ

PROPOSAL: 実行場所ごとに接続先を管理し、APIのhealth、依存先への到達性、実機能を段階別に検証する。healthの成功だけでは依存先との連携成功を示さない。

確認中にuvの親プロセスだけを中断してもUvicornがポートを保持し、WinError 10048が発生した。自分が起動したプロセスのIDとコマンドラインを確認して停止し、再起動した。これも接続先URLの問題とは分けて調べる。
