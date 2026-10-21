# uvのDockerタグを固定し、実際に使われる版を確認する

## はじめに

uvを更新する依頼に対し、PC上の実行ファイルとDockerfileの指定を別々に調べました。PCのuvはすでに0.13.4でしたが、コンテナはlatestを参照していました。この記事は、更新対象を特定してコンテナ内の版まで検証した記録です。

## 前提・環境

2026年10月22日、blog-wordcloudのanalysisサービスで確認しました。Windows上のDocker DesktopからLinuxイメージをビルドし、Pythonは既存の3.12.11を使っています。[Astralのリリース情報](https://github.com/astral-sh/uv/releases/tag/0.13.4)で確認した最新安定版は0.13.4でした。

## やってみた結果

Dockerfileの変更は次の1行です。uv.lockとPythonの依存宣言は変更しませんでした。

```diff
-COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/
+COPY --from=ghcr.io/astral-sh/uv:0.13.4 /uv /uvx /bin/
```

これは既知の旧版からの更新ではなく、移動するタグを特定のリリースへ固定する変更です。変更前のlatestが指していた実行版は測定していないため、実行ファイルの新旧比較は主張しません。

## 実装・検証

### 新しいイメージを実際にビルドする

実行場所はリポジトリルートです。ローカルDockerへ検証用イメージを作成します。

```powershell
docker build -t blog-wordcloud-analysis:uv-0.13.4 services/analysis
docker run --rm blog-wordcloud-analysis:uv-0.13.4 uv --version
```

ビルド中の `uv sync --frozen --no-cache` が成功し、イメージ内からuv 0.13.4が報告されました。ホストの `uv --version` だけでコンテナの版を確認済みとは扱いませんでした。

### 固定した版で既存テストも確認する

ビルド済みイメージへtestsを読み取り専用でマウントし、ダミーの必須設定を与えて15テストが成功しました。statement coverageは96.71%で、プロジェクトの80%基準を満たしました。具体的なテスト実行条件は[Docker内のpytest検証](./137-run-pytest-in-built-container.md)へ分けています。

## つまずいたところ

更新依頼があっても、すべてのuvを変更する必要があるとは限りませんでした。PC上のuvは変更せず、リポジトリ内の参照だけを修正しました。

## 学んだこと

タグの固定、依存インストールの成功、実行版の確認には、それぞれ別の証拠が必要でした。タグ固定の効果は版を明示できたことで、処理速度の改善は測定していません。

## 制約と根拠

根拠はblog-wordcloudのコミット `9fd1945d51eeb3ea6acf01dbec58db613cf942e4`、Dockerfile差分、ビルド出力、コンテナ内の版とテスト結果です。既存の本番コンテナを更新・デプロイしたわけではありません。digestによる固定は行っておらず、リリースタグの内容まで不変と保証するものではありません。

[既存の別事例](./120-uv-container-update-required-version.md)ではrequired-versionとの不整合が問題でした。今回はその設定がなく、版の固定とコンテナでの検証が対象です。
