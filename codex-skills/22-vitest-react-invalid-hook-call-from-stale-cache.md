# GitHub ActionsのVitestでReactのInvalid hook callが発生した原因と解決方法

## 結論

GitHub ActionsだけでVitestのブラウザテストが`Invalid hook call`になる場合、実行ツールのバージョンだけでなく、依存キャッシュに残ったVite最適化済みモジュールも確認する。今回はBunを固定した後も古い`frontend/node_modules`キャッシュが再利用されていたため、キャッシュキーへBunのバージョンを含めて再生成させた。

## 発生した問題

### 症状

```text
Invalid hook call.
TypeError: Cannot read properties of null (reading 'useContext')
```

`App`、`NotFoundView`、root routeのブラウザテストがCIで失敗し、テストが完了しないためカバレッジディレクトリも作成されなかった。

### 発生条件

- Environment: GitHub Actions、Ubuntu 24.04、Vitest browser mode、Chromium
- Initial runtime: `oven-sh/setup-bun@v2` の `bun-version: latest`（Bun 1.4.1）
- Local runtime: Bun 1.3.13
- Cache: `frontend/node_modules`をlockfileのハッシュだけで共有

## 調査

### 仮説1: カバレッジArtifactのパスが間違っている

Vitestの`reportsDirectory`は`./coverage`で、テストの作業ディレクトリは`frontend`だった。そのため、成功時の出力先は`frontend/coverage`であり、Artifactのパス自体は正しかった。

### 仮説2: CIとローカルのBunバージョン差

CIログはBun 1.4.1、ローカルはBun 1.3.13だったため、ワークフローのBunを1.3.13へ固定した。しかし、次のCIでも同じReactエラーが発生した。

### 仮説3: 古い依存キャッシュの再利用

Bunを固定した後もキャッシュキーがlockfileハッシュだけだった。`node_modules`内にはViteの最適化済み依存も含まれるため、実行環境を変更しても以前のキャッシュが復元される可能性があった。

## 原因

Bunバージョンを変更しても`frontend/node_modules`キャッシュを無効化していなかった。結果として、CIのブラウザテストが互換性のないReactモジュール構成を読み込み、`useContext`を呼び出す時点でInvalid hook callになった。

## 解決方法

フロントエンドの全CIワークフローで、node_modulesキャッシュキーにBunバージョンを含めた。

```yaml
key: ${{ runner.os }}-bun-1.3.13-${{ hashFiles('frontend/bun.lock') }}
```

カバレッジArtifactは、テスト失敗でレポートが生成されない場合に二次エラーを出さないよう、ファイルの存在を条件にアップロードする。

```yaml
if: ${{ !cancelled() && hashFiles('frontend/coverage/**') != '' }}
```

## 動作確認

ローカルの`frontend`で次を実行し、失敗は再現しなかった。

```text
bun run test
bun run test:small
bun run test:medium
bun run test:large
bun run test:storybook
bun run build
bun run lint
bun run typecheck
```

Smallは130件、Storybookは46件が成功した。SmallのカバレッジはStatements 96.62%、Branches 87.03%、Functions 96.29%、Lines 96.54%だった。

## 再発防止

- CIで`bun-version: latest`を使わず、検証済みバージョンを明示する。
- 実行ランタイムを変更したときは、依存キャッシュキーにもバージョンを含める。
- Artifactの存在しないことが先行ジョブの失敗結果である場合、Artifact処理が本来の失敗を隠さない条件にする。

## まとめ

- 症状: CIのブラウザテストだけがReactのInvalid hook callで失敗した。
- 原因: Bun変更後も古いnode_modulesキャッシュを再利用していた。
- 解決方法: Bunバージョンを固定し、キャッシュキーへ含めた。
- 教訓: CIの環境差はツールのバージョンとキャッシュの寿命を一緒に確認する。

確認日: 2026-09-05
