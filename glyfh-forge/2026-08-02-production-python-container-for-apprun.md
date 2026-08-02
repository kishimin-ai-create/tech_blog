# FastAPIをAppRun向けの本番コンテナにする

Glyph ForgeをさくらのクラウドのAppRun共用型へ載せるため、本番用Dockerイメージを作成した。開発環境をそのまま固めるのではなく、再現可能な依存関係、非root実行、固定ポートをコンテナの契約にした。

## マルチステージでwheelを作る

Dockerfileはbuilderとruntimeの二段構成で、どちらも`python:3.12.11-slim-bookworm`を使う。

builderでは、本番依存とGlyph Forge自身のwheelを`/wheels`へ作る。runtimeではそれらのwheelだけを`--no-index`でインストールし、ビルド用の作業ディレクトリを持ち込まない。

アプリをeditable installにしなかったのは、開発時のソース配置に依存しない、本番と同じ通常インストールを確認するためだ。パッケージ内のフォントもwheel経由で配置される。

## 本番依存を分離する

`requirements.txt`にはpytest、Black、flake8、mypyなど開発用ツールも含まれる。本番イメージでは`requirements-prod.txt`を別に用意し、API起動と画像生成に必要なものだけを固定バージョンで列挙した。

現在の本番依存はAnyIO、FastAPI、NumPy、Pillow、python-multipart、Uvicornである。依存を小さくする目的だけでなく、「本番で直接必要なパッケージは何か」をレビュー可能にする効果もある。

## UID 10001で実行する

runtimeにはUIDとGIDが10001の`app`ユーザーを作り、`USER app`へ切り替えた。APIは8080番ポートでUvicornを起動し、開発用の`--reload`は使わない。

起動コマンドは次の契約に固定した。

```text
uvicorn app.main:app --host 0.0.0.0 --port 8080
```

AppRunのヘルスチェックには既存の`GET /health`を使用できる。

## ビルドコンテキストも小さくする

`.dockerignore`では`.git`、テスト、キャッシュ、入力・出力画像、環境変数ファイル、開発用requirementsを除外した。イメージを軽くするだけでなく、ローカル成果物や認証情報候補を誤ってビルドコンテキストへ送らないための境界でもある。

## 関連する実装

- `Dockerfile`
- `requirements-prod.txt`
- `.dockerignore`
- `docs/sakura-cloud-deployment.md`
- コミット `bc81a53`、`8a2e29d`
