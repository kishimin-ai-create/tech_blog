# uvの指定バージョンだけをuvxで呼び出して検証を進める

## はじめに

プロジェクトが要求するuvと、WindowsのPATH上のuvが違い、検証コマンドを実行できなかった。グローバルのuvを更新せず、指定バージョンを`uvx`から呼び出して検証を進めた記録である。

## 前提・環境

- Windows / PowerShell、Python 3.14.7。
- PATH上のuvは0.12.15。
- `pyproject.toml`の`[tool.uv]`に`required-version = "==0.12.24"`がある。

## やってみた結果

```powershell
uvx --from uv==0.12.24 uv --version
```

実行結果は`uv 0.12.24`だった。そのuvで依存関係を同期し、Ruff、mypy、pytestを実行できた。`--from`によるパッケージ指定は[uv公式のツール実行ガイド](https://docs.astral.sh/uv/guides/tools/)でも説明されている。

## 実装・検証

### バージョン差を確認する

通常の`uv run`は、次のエラーで止まった。

```text
Required uv version `==0.12.24` does not match the running version `0.12.15`.
```

テストの失敗ではなく、ツールの起動条件に関する失敗だった。`required-version`を緩めたり、ロックファイルを変更したりせず、実行側を合わせた。

### 指定バージョンで既存コマンドを実行する

以下は対象プロジェクトのルートで実行した。初回は指定したuvパッケージの取得にネットワークが必要になる。

```powershell
uvx --from uv==0.12.24 uv sync --all-groups
uvx --from uv==0.12.24 uv run ruff format --check .
uvx --from uv==0.12.24 uv run ruff check .
uvx --from uv==0.12.24 uv run mypy app tests
uvx --from uv==0.12.24 uv run pytest acceptance tests --cov=app --cov-report=term-missing --cov-report=xml -q
```

最終的に整形・Lint・型チェックが成功し、196テストが成功した。API修正に伴う一時的なLint違反はコード側で直してから再検証した。uvの切り替えだけでコード品質の問題も解決したわけではない。

## つまずいたところ

Gitフックなど、内部から単に`uv`を呼ぶ処理には外側のコマンド指定だけでは伝わらない場合がある。今回は取得済みの0.12.24の実行ファイルがあるディレクトリを、そのシェルのPATH先頭へ置いた。ユーザー共通のPATHやuv本体は変更していない。フックを省略するオプションも使っていないが、コミットログだけを根拠にフックが実行されたとまでは断定しない。

キャッシュの実パスはマシン固有なので、共有手順へ固定値として転載しない。

## 学んだこと

ツールのバージョン不一致を、アプリケーションの失敗と区別する。プロジェクトの指定を維持したまま、作業中の実行環境だけを合わせられる場合がある。継続利用では、uvの共通更新方針やシェル起動時の解決先も別途整理する。

## 根拠と制約

根拠は実行ログ、`pyproject.toml`、`.github/workflows/ci.yml`、`.pre-commit-config.yaml`である。この方法を別のuvバージョンや別OSで検証したわけではない。グローバル更新は実施していない。
