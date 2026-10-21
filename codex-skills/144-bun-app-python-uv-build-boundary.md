# BunのアプリでもPythonサービスのビルドにuvが使われていた

## はじめに

「blog-wordcloudはBunで動かしているので、uvは使っていないのではないか」という確認に対し、アプリ本体と解析サービス、ソースと稼働コンテナを分けて調べた。調査日は2026年10月22日である。

## 前提・環境

- `app/backend`はBunとHonoを使用。
- `services/analysis`はPythonとFastAPIを使用。
- ローカルで`blog-wordcloud-api`コンテナが稼働。

## やってみた結果

uvを使っていたのは解析サービスのDockerビルドだった。APIの起動コマンドはFastAPIを直接実行しており、uvがAPIサーバーとして常時動いているわけではなかった。

| 確認対象 | 観測した内容 |
| --- | --- |
| Bunバックエンド | `bun run`で起動する設定 |
| 解析サービスのDockerfile | uvをコピーし、`uv sync`を実行 |
| 解析サービスのCMD | `.venv/bin/fastapi run ...` |
| 稼働コンテナ内のuv | 0.12.23 |

## 実装・検証

### ディレクトリ単位で利用箇所を読む

`app/backend/package.json`と`services/analysis/Dockerfile`を確認した。初回調査時点の後者には次の指定があった。

```dockerfile
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/
WORKDIR /services/analysis
RUN uv sync --frozen --no-cache
```

`pyproject.toml`と`uv.lock`も解析サービス配下にあった。リポジトリの主要言語だけでは、この依存インストール経路を判断できない。

### 稼働実体を確認する

```powershell
docker exec blog-wordcloud-api uv --version
docker inspect blog-wordcloud-api --format '{{json .Config.Cmd}}'
docker top blog-wordcloud-api -eo pid,comm
```

uvは0.12.23で、起動設定とプロセスはFastAPIだった。コンテナの作成日は10月4日である。Dockerfileに`latest`と書かれていても、この既存コンテナへ後日のuvが自動適用されたわけではなかった。

## つまずいたところ

最初の説明では「blog-wordcloudでuvを使っている」とリポジトリ全体をまとめて表現した。ビルド時の使用とリクエスト処理中の実行を分けると、Bunを使っているという理解と矛盾しない。

## 学んだこと

バージョン調査では、コマンドが存在すること、Dockerビルドで実行されること、起動中プロセスであることを分ける。古いuvがイメージに含まれているという事実だけで、APIの全通信がそのuvのCAストアを使っているとは判断できない。

この調査では解析サービスの更新や再作成をしていない。その後の執筆中に別作業のコミット`9fd1945`でDockerfileが0.13.4へ固定されたことを確認したが、稼働コンテナ内のuvは再確認でも0.12.23だった。ソースの指定と稼働実体はここでも異なっていた。通信経路全体の脆弱性評価や本番の構成確認は実施していない。

## 参考資料

- 根拠：`app/backend/package.json`、`services/analysis/Dockerfile`、`services/analysis/pyproject.toml`、ローカルコンテナの起動設定と版。
- [Dockerイメージを定期的に再ビルドする際の注意](https://docs.docker.com/build/building/best-practices/)
