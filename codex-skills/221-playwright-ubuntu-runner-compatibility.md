# Ubuntu の更新後に Playwright のブラウザー導入が失敗した原因

## 結論

GitHub Actions の `ubuntu-latest` が Ubuntu 26.04 を選択した実行で、Playwright 1.60.0 は Chromium の依存パッケージを導入できなかった。PR と Nightly のフロントエンドジョブを `ubuntu-24.04` に固定したところ、PR CI ではブラウザー導入と E2E が成功した。修正後の Nightly と VRT は未確認である。

## 発生した問題

### 症状

[Nightly のログ](https://github.com/kishimin-ai-create/diary/actions/runs/41932156218/job/126931477061)で、Playwright は `ubuntu26.04-x64` 向けの依存関係を導入できず、Chromium がその環境でサポートされないと報告した。

### 発生条件

- GitHub Actions の `ubuntu-latest` が Ubuntu 26.04 を選択
- Playwright 1.60.0
- `bunx playwright install --with-deps` でブラウザーと OS 依存パッケージを導入

### 影響

Nightly はブラウザー導入で停止し、その後の E2E と VRT には到達しなかった。

## 調査

### 仮説1：Linux のバージョン変更が導入失敗に関係する

失敗したジョブのログと workflow の `runs-on` を確認した。ログは Ubuntu 26.04 と Playwright のサポート外を示し、workflow は `ubuntu-latest` を指定していた。PR のフロントエンドジョブにも Playwright 導入ステップがあることを確認した。

## 原因

`ubuntu-latest` は固定された OS バージョンではない。今回選択された Ubuntu 26.04 に対し、ログに現れた Playwright 1.60.0 の依存関係導入は対応しておらず、ブラウザー導入ステップがエラー終了した。

## 解決方法

[PR #10](https://github.com/kishimin-ai-create/diary/pull/10) で、PR と Nightly のフロントエンドジョブの runner を固定した。

```yaml
runs-on: ubuntu-24.04
```

これは検証済みのジョブ環境を明示する対応である。Playwright 自体の更新や、他の OS への対応を確認したものではない。

## 動作確認

[修正後の PR CI](https://github.com/kishimin-ai-create/diary/actions/runs/42017512803) ではブラウザー導入が完了し、E2E は 2 件成功した。PR はマージされた。修正後の Nightly は未確認であり、この結果だけで Nightly の VRT 成功までは判断できない。ローカルの YAML 読み込みと `git diff --check` は成功したが、Prettier check は両 workflow の書式警告を報告した。

## 再発防止

OS や Playwright を更新するときは、その組み合わせでブラウザー依存関係の導入と実際のテストを確認する。VRT は OS 変更による描画差もあるため、次の Nightly で差分を確認する。

## まとめ

ランナーの可変ラベルを使う CI では、アプリケーションの変更がなくても OS とブラウザーツールの対応関係が変わり得る。失敗ログの OS と Playwright のバージョンを照合し、固定後も対象の CI が最後まで進むことを個別に確認する。
