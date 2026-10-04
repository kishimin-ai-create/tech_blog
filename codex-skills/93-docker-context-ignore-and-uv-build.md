# uvでFastAPIをDocker化するときの配置とビルドコンテキスト

## はじめに

FastAPIの画像生成サービスをDocker化し、ビルドとhealth応答を確認した。この記事ではDockerfileの場所、コピー元、除外設定の位置を混同せずに確認する方法を説明する。

## 前提・環境

- Windows上のDocker Desktop、Linuxコンテナ
- Pythonベースイメージ: 3.12.11-slim-bookworm
- サービスの依存定義: pyproject.toml、uv.lock
- 確認日: 2026年10月4日

## やってみた結果

FACT: Dockerの静的チェックは警告なし、イメージのビルドは成功した。既定コマンドで起動したコンテナのhealthがHTTP 200を返した。実際の画像生成やcollector連携はこのビルド検証では行っていない。

## 実装・検証

### 配置とコピー元を揃える

最終配置は次の形になった。

```text
services/analysis/
├── Dockerfile
├── .dockerignore
├── pyproject.toml
├── uv.lock
├── main.py
├── analyze.py
└── analysis/
```

リポジトリルートから、Dockerfileとコンテキストを明示する。

```powershell
docker build -f services/analysis/Dockerfile -t blog-wordcloud-analysis:local services/analysis
```

このコマンドは実行して成功した。COPYのコピー元は末尾のコンテキストで決まり、Dockerfileが置かれている場所だけでは決まらない。

### 依存関係と起動を設定する

実際のDockerfile:

```dockerfile
FROM python:3.12.11-slim-bookworm
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/
COPY . /services/analysis
WORKDIR /services/analysis
RUN uv sync --frozen --no-cache
CMD [".venv/bin/fastapi", "run", "main.py", "--port", "80", "--host", "0.0.0.0"]
```

FACT: uv.lockをコミット `8aed417`、Dockerfileと除外設定を `fc2c2b2` に分けた。uv lock --checkも成功した。frozenはロックの更新と最新性チェックを省略するため、別に整合を確認した。

### コピー対象を除外する

コンテキスト直下の.dockerignoreで、ローカルの.venv、.envと.env.*、tests、キャッシュ、input、output、scriptsを除外した。フォントを含むanalysis/assetsはコピー対象に残る。outputはハンドラがmkdirで作成する。

通常の.dockerignoreはコンテキストのルートへ置く。Dockerfileと同じ階層に専用の除外ファイルを置く方式には、Dockerfile名を接頭辞に付ける別の命名規則がある。[Docker公式の配置ルール](https://docs.docker.com/build/concepts/context/#filename-and-location)

## つまずいたところ

以前の起動パスはWORKDIRに対してディレクトリが重複していた。.dockerignoreもコンテキストと別の階層へ置いていた。レビューでは、コンテキストを明記せず配置だけを指摘したため説明が分かりにくくなった。最終的にコンテキストを前提として、COPY・WORKDIR・CMDの対応を確認した。

また、起動後2秒でのhealth確認は接続拒否だった。準備完了を待って再確認するとHTTP 200になった。固定時間の失敗だけを起動不能と判断しない。

## 学んだことと制約

INFERENCE: 配置の正しさはビルドコマンドとセットで判断する。uv公式はローカル仮想環境の除外を推奨するが、推奨と必須の構文条件を混同しない。[uvのDocker統合](https://docs.astral.sh/uv/guides/integration/docker/#installing-a-project)

現在の設定はuvのlatestタグを使い、開発依存もインストールする。uvの固定と開発依存除外は未実施であり、再現性やイメージ容量の改善まで完了したとは言えない。

## まとめ

実際のビルドとhealth検証は成功した。次に確認すべきものは実collector連携と画像生成である。ローカルファイルの位置だけでなく、コンテキストからコピー後までを追うことが今回の学びだった。
