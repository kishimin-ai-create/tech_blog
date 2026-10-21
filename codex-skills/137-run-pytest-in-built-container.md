# ビルド済みDockerイメージでpytestを動かすための前提を揃える

## 結論

Docker内で既存テストを再実行する際、テストファイル、Pythonのimport経路、必須環境変数の3つを揃えました。アプリのソースは修正せず、ビルド済みイメージに対して15テストが成功しました。

これは2026年10月22日、blog-wordcloudのanalysisサービスをuv 0.13.4で検証した際の記録です。同じように、ホストでは動くテストをDocker内で再現したい人を対象としています。

## 発生した問題

Dockerfileは `uv sync --frozen --no-cache` まで成功していました。しかし、テストを実行するにはビルド成功だけでは足りませんでした。

| 実行条件 | 観測した結果 |
| --- | --- |
| .dockerignore | testsをビルド対象から除外 |
| testsをマウントして直接pytest | analyze・main・analysisのModuleNotFoundError |
| python -m pytestへ変更 | 設定のimportまで進み、SLOPE_COLLECTOR_URLのKeyError |
| 必須設定にダミー値を付与 | 15テスト成功 |

実行環境はDocker DesktopのLinuxコンテナ、Python 3.12.11、pytest 7.4.4、pytest-cov 7.0.0です。

## 調査

### テストを本番用ビルドへ混ぜずに渡す

.dockerignoreはtestsを除外しています。そこでソースやDockerfileのCOPY対象を変更せず、実行時にtestsを読み取り専用でマウントしました。

### Pythonのモジュール探索を確認する

直接のpytest呼び出しでは、テストからアプリモジュールをimportできませんでした。ビルド済みイメージの作業ディレクトリ `/services/analysis` から `python -m pytest` を使うと、これらのimportは通りました。この差は今回の環境で観測したもので、すべてのpytest起動が同じ結果になるとは限りません。

### import時に要求される設定を確認する

次に止まったのはanalysis/config.pyでした。このファイルはimport時にos.environからcollectorのURLと3種類の保護語を取得します。コンテナには.envをコピーしないため、テスト用の設定を明示する必要がありました。

## 解決方法

次のPowerShellコマンドをリポジトリルートで実行しました。Dockerへソース内のtestsを読み取り専用で公開し、一時コンテナだけにダミー設定を渡します。

```powershell
$analysisTests = (Resolve-Path services/analysis/tests).Path
docker run --rm `
  -e SLOPE_COLLECTOR_URL=http://collector.test `
  -e O_MEET_PROTECTION_WORD=sample-one `
  -e R_MEET_PROTECTION_WORD=sample-two `
  -e MEET_PROTECTION_WORD=sample-three `
  --mount "type=bind,source=$analysisTests,target=/services/analysis/tests,readonly" `
  blog-wordcloud-analysis:uv-0.13.4 `
  uv run --frozen python -m pytest --cov --cov-report=term-missing
```

実際の検証では同じtestsディレクトリの絶対パスを指定しました。上記では別PCでも解決できる表記へ置き換えています。イメージのビルド手順は[uvの固定と実行版確認](./136-pin-uv-and-verify-container-version.md)を参照してください。

実際の.envや秘密値は渡していません。画像生成テストではHTTPアクセス等をモック化しており、collector.testとの実通信を検証するコマンドではありません。

## 動作確認

| 項目 | 結果 |
| --- | --- |
| pytest | 15 passed |
| Statement coverage | 206 / 213、96.71% |
| 設定された最低基準 | 80%、達成 |

カバレッジはpyproject.tomlの既存設定を使用しました。pytestは既存の4ファイルを収集し、import時の設定不足による停止は解消しました。

## 再発防止として考えたこと

提案として、テスト起動の標準コマンドに作業ディレクトリとダミー設定を含めると、同じ前提を再現しやすくなります。今回、それを新たなscriptやCIへ実装したわけではありません。

## 根拠と制約

根拠はservices/analysisの.dockerignore、analysis/config.py、pyproject.toml、既存テストと3段階の実行結果です。uv更新後のアプリ検証で見つけた実行条件であり、旧uvと新uvの比較実験ではありません。実サービスとの結合、本番環境での画像生成、カバレッジ成果物の永続保存までは行っていません。
