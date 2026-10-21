# uvのイメージ更新と稼働コンテナへの適用を分けて確認する

## はじめに

slope-collectorのuvを0.13.4へ更新した際、Dockerビルドの成功後も、稼働中APIには0.12.24が残っていた。2026年10月22日にローカルの開発APIを切り替え、DBコンテナを維持したことまで確認した。

## 前提・環境

- Windows上のDocker Desktop、Linuxコンテナ。
- APIとMySQLをComposeで起動。
- Composeは`.env.development`を読み、APIは`127.0.0.1:8080`へ公開。
- 対象は開発環境であり、本番環境ではない。

## やってみた結果

新しいAPIコンテナ内でuv 0.13.4を確認した。Dockerのhealth状態はhealthyで、ホストからの`GET /health`は`{"status":"ok"}`を返した。切り替え前後でDBコンテナIDは同じだった。

## 実装・検証

### イメージを検証する

```powershell
docker build --tag slope-collector:uv-security-check .
docker run --rm --entrypoint uv slope-collector:uv-security-check --version
```

ビルドと一時コンテナでの確認は成功し、uv 0.13.4が出力された。一方、既存の`slope-collector-api-1`で`uv --version`を実行すると0.12.24だった。新しいイメージの存在だけでは稼働中コンテナは更新されない。

### 起動条件と退避先を確認する

`docker inspect`のComposeラベルから、現在の設定ファイルと作業ディレクトリを確認した。Composeが解決するAPI環境変数と既存コンテナの設定を値を公開せず照合し、一致を確認した。APIプロセスはUvicornだった。

旧APIイメージには退避用タグを付け、新しい検証済みイメージをComposeのAPIイメージ名へタグ付けした。旧タグを残したことは復旧材料の保存であり、ロールバックの動作試験を実施したという意味ではない。

### APIだけを再作成する

以下は実際に使った適用コマンドである。事前のタグ付けを前提とし、APIの一時停止を伴う。

```powershell
docker compose --env-file .env.development up -d --no-deps --no-build --wait --wait-timeout 60 api
```

`--no-deps`で依存サービスの再作成を要求せず、`--no-build`で検証済みイメージを利用した。その後、API内のuv、health状態、HTTP応答、DBコンテナIDを別々に確認した。

## つまずいたところ

最初の報告ではホストとDockerfileの更新範囲を説明していたが、稼働中の版と本番への反映を十分に分けていなかった。実行中のコンテナへ確認に入ると旧版が残っていたため、利用者の承認後にAPIを再作成した。

## 学んだこと

設定変更、イメージ作成、一時コンテナでの確認、稼働サービスへの適用は別の結果である。DBコンテナIDの一致は再作成していないことの証拠になるが、DB全件の内容照合を行った証拠にはならない。今回はデータ更新やマイグレーションは行っていない。

ローカルのhealthyとHTTP成功は、本番への反映や全業務経路の正常性まで保証しない。本番環境の更新は今回実行していない。

## 参考資料

- [今回の変更コミット](https://github.com/kishimin/slope-collector/commit/2d8b1905fb104b30bf333945e7eddd33a3d5c8b3)
- [Dockerのビルドに関するベストプラクティス](https://docs.docker.com/build/building/best-practices/)
