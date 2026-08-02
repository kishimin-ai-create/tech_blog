# Dockerfileの本番契約をテストで守る

コンテナ設定はコードではないが、運用上の重要な仕様を持つ。ベースイメージ、実行ユーザー、ポート、起動コマンド、含める依存が変われば、本番の挙動も変わる。

Glyph ForgeではDockerfileを追加する前に、`tests/test_glyph_forge/test_container_config.py`で本番コンテナの契約を定義した。

## 何をテストしたか

テストは三つの観点を持つ。

### 本番依存

`requirements-prod.txt`の内容を固定し、開発用ツールが混ざっていないことを確認する。依存追加は明示的なレビュー対象になる。

### 実行条件

Dockerfileが固定したPython 3.12.11 slimイメージを使い、非rootの`app`ユーザーへ切り替え、8080番でAPIを起動することを確認する。`--reload`やeditable installが含まれないことも見る。

### ビルドコンテキスト

`.dockerignore`がGit履歴、テストキャッシュ、Pythonキャッシュ、テスト、ローカル画像、`.env`、開発用requirementsを除外することを確認する。

## 文字列検査の限界

このテストはDockerfileを実際にビルドする代わりにはならない。構文が正しくても、ベースイメージ取得、wheelのビルド、アーキテクチャ、起動時import、画像生成まで成功する保証はない。

実際、設定テストを満たした最初の本番依存にはNumPyがなく、画像生成のスモークテストで不足が分かった。

したがって役割分担は次のようになる。

- 設定テスト: 意図した契約からの差分を高速に検出する
- Docker build: イメージを組み立てられることを確認する
- スモークテスト: 起動と主要機能を確認する

設定ファイルにもTDDを適用できるが、実環境でしか見えない問題を補う検証層は残しておく必要がある。

## 関連する実装

- `tests/test_glyph_forge/test_container_config.py`
- `Dockerfile`
- `.dockerignore`
- `requirements-prod.txt`
- コミット `bc81a53`、`8a2e29d`、`08983ff`
