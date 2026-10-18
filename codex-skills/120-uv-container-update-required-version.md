# QualityとTestが成功してもDockerのuv更新が失敗した理由

## 結論

Dependabotによるuvイメージの更新で、Dockerfileだけが0.13.1になり、`pyproject.toml`の要求は0.12.24のまま残っていた。コンテナの`uv sync --frozen`は要求バージョンとの不一致で停止した。更新先に合わせて`required-version`も0.13.1にすると、ローカルのDockerビルドは成功した。

重要だったのは、同じPRのQualityとTestが成功していたことだ。この2ジョブは`pyproject.toml`から0.12.24を選択しており、Dockerの0.13.1とは実行環境が違っていた。

## 発生した問題

対象はslope-collectorの[PR #36](https://github.com/kishimin/slope-collector/pull/36)。2026年10月18日に確認したCI run `38813344494`では、Quality・Test・systemd unitsは成功し、Containerだけが失敗していた。

```text
Required uv version `==0.12.24` does not match the running version `0.13.1`
```

失敗した場所はDockerfileの`RUN uv sync --frozen --no-dev --no-install-project`。依存ライブラリのテストまで進む前に、ツール自身の実行条件に抵触していた。

### 発生条件

| 設定・実行箇所 | uvのバージョン |
| --- | --- |
| Dockerfileの`FROM ghcr.io/astral-sh/uv` | 0.13.1 |
| `pyproject.toml`の`required-version` | `==0.12.24` |
| Qualityジョブのsetup-uvログ | 0.12.24を検出・インストール |

GitHubのContainerジョブはUbuntu 24.04で実行されていた。ローカルの再現はWindows上のDocker DesktopのLinuxコンテナで行った。

## 調査

最初にrunの結果を確認し、失敗したContainerジョブのログを読んだ。ログには、uvイメージ0.13.1の取得と、要求0.12.24との不一致が続けて記録されていた。

次に、成功していたQualityジョブのログも確認した。そこには`pyproject.toml`からuv 0.12.24を検出し、そのバージョンをインストールした記録があった。成功したジョブで、新しいDocker内のuvまで検証したわけではなかった。

PR専用worktreeで修正前の`docker build`も実行した。同じエラーで失敗し、CI上だけの一時的な通信問題ではないことを確認した。

## 原因

uvの選択元が2箇所あった。CIのホスト側はプロジェクト設定を読み、Dockerはイメージタグで選んだ実行ファイルを使う。Dependabotの差分はDockerfileの1行だったため、両者の整合性が崩れた。

`--frozen`はロックファイルを使うための指定であり、要求されたuvバージョンの検査を無効化するための指定ではない。今回の実行結果でも、ロック済み依存関係を利用する前にバージョン検査で停止した。

## 解決方法

更新対象のPRブランチで、プロジェクトが要求するuvも更新先へ揃えた。

```diff
 [tool.uv]
-required-version = "==0.12.24"
+required-version = "==0.13.1"
```

制約を削除せず、Dockerイメージを元へ戻すこともしなかった。依存パッケージの更新は行わず、`uv.lock`にも変更はなかった。

## 動作確認

プロジェクトルートで、更新先のuvを`uvx`から呼び出した。バージョン指定の基本操作は[前回の記事](./118-project-uv-version-with-uvx.md)にまとめている。

```powershell
uvx --from uv==0.13.1 uv sync --all-groups --frozen
uvx --from uv==0.13.1 uv run ruff format --check .
uvx --from uv==0.13.1 uv run ruff check .
uvx --from uv==0.13.1 uv run mypy app tests
$env:DATABASE_URL = 'mysql+pymysql://db/collector'
uvx --from uv==0.13.1 uv run pytest acceptance tests --cov=app --cov-report=term-missing --cov-report=xml -q
docker build --tag slope-collector:pr36-verified .
```

別worktreeにはローカルの`.env`がないため、最初のpytestは設定不足でcollectionに失敗した。再実行ではCIと同じ、認証情報を含まない`DATABASE_URL`をプロセスに設定した。これで実DBへ接続したという意味ではなく、テストが起動時に必要とする設定を満たしたものである。

| 検証 | 結果 |
| --- | --- |
| 整形 | 63ファイル成功 |
| Lint | 成功 |
| 型チェック | 44ソースファイル成功 |
| テスト | 188件成功 |
| 行／分岐／総合カバレッジ | 91.25%／84.13%／89.88% |
| Dockerビルド | 成功 |
| 新しいイメージの実行確認 | uv 0.13.1、Python 3.14.7 |
| 起動完了後の`/health` | `{"status":"ok"}` |

ヘルス確認には一時コンテナを使い、既存の開発DBや稼働APIを入れ替えていない。初回は起動直後にアクセスして接続拒否になったため、再確認ではDockerのhealthy状態を待ってから実際のHTTP応答を取得した。一時コンテナは停止・削除した。

## 再発防止と残る確認

既存のContainerジョブが今回の不整合を検出できていたため、同じ値を比較するだけの新規テストは追加しなかった。依存ツールを更新するときは、イメージタグだけでなく、それを利用するプロジェクトの要求も確認する。

この時点で確認できたのは修正前のCI失敗と、修正後のローカル検証である。修正コミットのPR反映と、そのSHAでのCI成功は公開後に別途確認する。別PRのPython更新や、アプリケーションのAPI修正は混ぜていない。

## 根拠

- [失敗したCI run 38813344494](https://github.com/kishimin/slope-collector/actions/runs/38813344494)
- [PR #36](https://github.com/kishimin/slope-collector/pull/36)のDockerfile差分
- ローカル修正コミット：`c66655d318de8cf8bf37e25eb6a2c437168f403a`
- 対象：`Dockerfile`、`pyproject.toml`、`.github/workflows/ci.yml`
