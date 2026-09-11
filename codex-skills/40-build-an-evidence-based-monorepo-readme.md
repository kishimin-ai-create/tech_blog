# 実ファイルを根拠にモノレポのREADMEを整備する

## 対象読者

ReactフロントエンドとASP.NET Core APIを同じリポジトリで管理し、初めて参加する開発者がREADMEだけで環境構築・API確認・テスト実行へ進める状態を作りたい開発者を対象にする。

## スコープ

MojicaのREADMEを、依存定義・CI・API実装・Dockerfile・ディレクトリ構成から確認できる情報だけで整備した作業を扱う。READMEの一般的な文章術や、実装コードそのものの変更は扱わない。

## 結論

モノレポのREADMEは、プロジェクト紹介だけでなく、実際に動かすための参照文書として作る必要がある。Mojicaでは、フロントエンドの`package.json`、バックエンドの`.csproj`、Playwright設定、CIワークフロー、APIルート、Dockerfileを突き合わせ、環境、起動手順、API契約、コマンド、既知のトラブルを一つの導線へまとめた。

## 作業前の課題

変更前のREADMEは、リポジトリ名と短い説明だけだった。その状態では、次の情報をリポジトリ内から別々に探す必要がある。

- フロントエンドとバックエンドがどの役割を持つか
- 必要なランタイムと依存バージョン
- ローカル起動時のポートとAPI URL
- APIの成功・失敗レスポンス
- Small・Medium・Large・Storybook・E2Eの実行コマンド
- CIやE2Eで接続失敗が起きた場合の確認方法

READMEに書く情報を想像で補うと、実装やCIと異なる手順を案内する危険がある。そのため、先に根拠となるファイルを確認した。

## 根拠を集める

READMEの各記述を、次のファイルと対応付けた。

| READMEに記載した情報 | 確認した根拠 |
| --- | --- |
| Bun・TypeScript・React・Viteのバージョン | `frontend/package.json`、`frontend/bun.lock` |
| .NETとASP.NET Coreの対象 | `backend/Mojica.Api/Mojica.Api.csproj` |
| フロントエンドのコマンド | `frontend/package.json`の`scripts` |
| APIの`/health`と`/images` | バックエンドのルーティングとOpenAPI設定 |
| 5ブラウザープロジェクトのE2E | `frontend/playwright.config.ts`、`frontend/e2e` |
| APIのポート、Glyph Forge、レート制限 | `.github/workflows/ci-*.yml`、バックエンド設定 |
| コンテナのポートとヘルスチェック | `backend/Dockerfile`、CIワークフロー |
| リポジトリの責務分担 | `backend`、`frontend`、`docs`、`.github/workflows`の実ディレクトリ |

この対応付けを先に作ると、READMEに記載する値と、その値が変更されたときに確認すべきファイルが明確になる。

## READMEの構成

READMEは、読者が上から順に進める構成にした。

1. プロジェクトの役割と構成
2. 必要な環境
3. ディレクトリ構成
4. インストールと起動
5. APIの最小利用例
6. エンドポイントとリクエスト制約
7. 実在するコマンド一覧
8. 実際に確認したトラブルシュート

特にAPIについては、成功時の`200 OK`と`image/png`だけでなく、入力不備、バリデーション、レート制限、Glyph Forge障害、タイムアウトに対応するステータスも記載した。利用者は成功例よりも、失敗したときに何を調べればよいかでREADMEの価値を判断するためである。

## 実行手順を実ファイルへ合わせる

フロントエンドのコマンドは`frontend/package.json`に存在するスクリプトだけを載せた。

```text
bun run dev
bun run build
bun run typecheck
bun run lint
bun run test:small
bun run test:medium
bun run test:large
bun run test:storybook
bun run e2e
```

バックエンドについては、ソリューションの復元・ビルド・テスト・カバレッジ取得を、CIワークフローで実際に使われているコマンドに合わせた。README独自のラッパースクリプトを追加するのではなく、読者がCIと同じコマンドをローカルで再実行できることを優先した。

## Troubleshootingを実際の症状から書く

今回のREADMEには、実際に観測した接続エラーと、リリースE2Eの環境変数未設定によるスキップを載せた。

```text
curl: (7) Failed to connect to 127.0.0.1 port 5063
```

このエラーに対しては、CIと同じ`ASPNETCORE_URLS`でAPIを起動する手順を示した。また、リリースE2Eについては、`RELEASE_E2E_BASE_URL`がない場合に意図的にスキップされることと、設定例を記載した。

想像上のエラーを増やさず、検索可能な実際のエラー文字列を見出しにしたことがポイントである。

## 動作確認

README整備はMojicaのコミット`21fe5a4 docs: document repository setup`として行った。差分はREADMEだけであり、コード・設定・依存関係は変更していない。

記事作成時点で確認できた事実は次のとおり。

- `frontend/package.json`のスクリプトとREADMEのAvailable Commandsが対応している。
- `.github/workflows`にSmall、Medium、Large、Storybook、E2E、Coverageの処理が定義されている。
- READMEのディレクトリ構成に記載した主要ディレクトリが存在する。
- READMEにはTable of Contents、Environment、Getting Started、API Endpoints、Troubleshootingが含まれている。

READMEに記載したすべてのコマンドをこの記事作成時に再実行したわけではないため、実行結果と根拠ファイルの照合は分けて扱う。

## 事実・判断・未確認事項

- FACT: READMEは`21fe5a4`で、短い説明から環境・起動・API・コマンド・トラブルシュートを含む構成へ更新された。
- FACT: フロントエンドのコマンドは`frontend/package.json`に定義されている。
- FACT: CIワークフローにはAPI、Frontend、Storybook、E2E、Coverageの処理がある。
- FACT: APIは`/health`と`/images`を公開し、画像生成成功時はPNGを返す。
- INFERENCE: READMEを実ファイルとCIから組み立てると、導入手順とCI手順の乖離を減らしやすい。
- ASSUMPTION: READMEの記載を将来も正確に保つには、依存バージョン、スクリプト、API契約、CI変更時にREADMEを再確認する必要がある。
- 未確認: READMEに記載したすべてのコマンドを、記事作成時にクリーン環境で一通り実行したわけではない。

## まとめ

README整備の中心は文章を増やすことではなく、読者が必要とする情報を実ファイルへ結び付けることだった。依存定義、CI、API、Dockerfile、ディレクトリ構成を根拠にして、導入・利用・検証・障害対応までの導線を作ると、READMEをリポジトリの参照文書として運用できる。

## 参考

- `mojica/README.md`
- `mojica/frontend/package.json`
- `mojica/frontend/playwright.config.ts`
- `mojica/backend/Mojica.Api/Mojica.Api.csproj`
- `mojica/.github/workflows/ci-pull-request.yml`
- `mojica/.github/workflows/ci-push.yml`
- `mojica/.github/workflows/ci-nightly.yml`
- Mojica commit `21fe5a4`

確認日: 2026-09-06
