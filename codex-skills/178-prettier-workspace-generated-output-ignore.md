# workspace内のPrettierが生成物を処理するときは実行場所から除外を確認する

## 結論

リポジトリルートにignoreファイルがあっても、workspace内から実行したPrettierへ同じ除外が適用されるとは限らない。profile-hubでは、公開アプリ側の実行場所に生成物の除外を置き、ビルド後にも整形チェックが通ることを確認した。

確認日は2026年10月27日。Bun workspaceとPrettier 3.9.6を使った実例である。対象の変更は未マージ。

## ビルド後に生成ファイルが対象になった

公開アプリの`format`と`format:check`は、それぞれ`prettier --write .`と`prettier --check .`だった。`apps/public-web`から実行すると、ビルド済みの`dist`内ファイルが対象になった。Storybookの生成後には、その生成物も整形チェックに入った。

この観測から分かるのは、整形対象の範囲に生成物が含まれたことである。Prettierの整形アルゴリズムに不具合があると判断する根拠にはならない。

## 実行場所と除外ファイルを合わせた

[公式のIgnoring Code](https://prettier.io/docs/ignore)でignoreの読み込みと`--ignore-path`を確認した。ルートの設定が自動的にすべての実行場所へ引き継がれることを期待せず、公開アプリに`.prettierignore`を追加した。

```text
node_modules/
dist/
coverage/
storybook-static/
playwright-report/
test-results/
```

ソースを除外せず、生成先を明示する。生成先が増えた場合は、ビルドやテストの出力設定と一緒に見直す必要がある。

## どこから何を確認したか

アプリ側の除外を設定した後、ビルド済みの生成物がある状態でも整形チェックを確認した。最終確認では、リポジトリルートから既存のscriptに引数を渡した。

```powershell
bun run format:check --ignore-path .gitignore
```

これはルートの`.gitignore`を明示して確認したコマンドであり、各workspaceからの実行と同じ条件だとは扱わない。設定を調べるときは、コマンド名だけでなく実行ディレクトリも記録すると再現しやすい。

## 適用するときの注意

除外を広げてソースの違反まで見えなくすると、整形チェックの目的を失う。今回の対処は生成先の除外に限定した。チェックを通すために生成物を手で整形する運用も採っていない。

別のworkspaceでも同じ問題があるとは未確認である。失敗したときは、まず実際の対象ファイルと作業ディレクトリを確認し、必要な場所へ除外を置く。

## 根拠

- [対象コミットのpublic-web用ignore](https://github.com/kishimin-ai-create/profile-hub/blob/4457f6f8624782d80bb8196a4c00c5319cf24325/apps/public-web/.prettierignore)
- [公開アプリのscript定義](https://github.com/kishimin-ai-create/profile-hub/blob/4457f6f8624782d80bb8196a4c00c5319cf24325/apps/public-web/package.json)
