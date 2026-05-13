# 2 つの GitHub アクション ワークフローのバグが修正されました: MySQL サービス コンテナーと `node --env-file` の欠落

**タイプ:** 技術的な問題の解決策
**コミット:** `98a7490`、`3087fd6`
**対象読者:** GitHub Actions でライブ MySQL データベースに対して Vitest 統合テストを実行しているエンジニア、または Node.js を使用してデプロイ ワークフローからリモート サーバー上で移行スクリプトを実行している人。

---

## 概要

別々の GitHub Actions ワークフローの 2 つのバグが独立して CI の適用範囲を破っていました
実行と実稼働データベースの移行。彼らは同じ種類の失敗の出身です。
**データベース資格情報またはインフラストラクチャに依存するワークフロー ステップは、これらを前提としています。
前提条件はすでに整っていたのですが、そうではありませんでした。**

この投稿では、各バグ、何が失敗したか、その理由、および最小限の正しい修正を文書化します。

---

## 修正 1 — MySQL サービス コンテナが `test-coverage.yml` から欠落している

### 症状

`Test Coverage` ワークフローのすべての実行は次の時点で終了しました。

```
Fail when coverage failed
exit 1
```

`Run backend integration coverage` ステップは一貫して失敗としてマークされていましたが、
出力には目に見えるテスト アサーション エラーはありませんでした。テストランナーはできませんでした
開始する前に何かに接続します。

### 根本的な原因

`backend/vitest.integration.config.ts` は、`src/tests/integrations/` に基づくテストをターゲットとします。
これにより、ライブ MySQL 接続が開きます。ワークフローは `npm run coverage:integration` を実行しました
`services:` ブロックなし - MySQL コンテナが起動されていないことを意味します - そして
`DB_*` 環境変数を設定せずに。 mysql2 接続プールが失敗しました
実行するたびにすぐに。

永続的な赤色を引き起こすパターン:

```yaml
# Before — no services block on the job, no DB env vars on the step
- name: Run backend integration coverage
  id: backend_integration_coverage
  continue-on-error: true        # failure is caught, not propagated immediately
  run: npm run coverage:integration
  working-directory: backend
  # ← DB_HOST, DB_PORT, DB_DATABASE, DB_USERNAME, DB_PASSWORD all undefined

- name: Fail when coverage failed
  if: steps.backend_integration_coverage.outcome == 'failure'
  run: exit 1                    # fires every single time
```

`continue-on-error: true` はここでは意図的です。これにより、カバレッジ アーティファクトもアップロードできるようになります。
閾値を超えたとき。しかし、MySQL 接続が欠落しているため、実際の接続は存在しませんでした。
カバレッジ結果をアップロードします。 `exit 1` ゲートは無条件に起動されました。

### 修正 (`98a7490` をコミット)

`.github/workflows/test-coverage.yml` に 3 つのことが追加されました。

**1.ジョブ上の `services.mysql` ブロック:**

```yaml
services:
  mysql:
    image: mysql:8.0
    env:
      MYSQL_ROOT_PASSWORD: ""
      MYSQL_ALLOW_EMPTY_PASSWORD: "yes"
      MYSQL_DATABASE: TDDTodoAppDB
    ports:
      - 3306:3306
    options: >-
      --health-cmd="mysqladmin ping -h 127.0.0.1 --silent"
      --health-interval=10s
      --health-timeout=5s
      --health-retries=5
```

GitHub Actions サービス コンテナは、ジョブ ランナーと同じネットワーク名前空間を共有します。
MySQL コンテナは、マップされたポート上の `127.0.0.1` でアクセス可能です (ホスト名なし)
解像度またはブリッジネットワークが必要です。

`--health-cmd` ブロックは重要です。これがないと、後続のステップが先に開始される可能性があります。
MySQL は初期化完了後に接続を受け付け始めるため、タイミング次第で見かけ上の
`connection refused` エラー。 GitHub Actions はヘルスチェックに合格するまで待機します
`options` がこのように構成されている場合、最初のステップに進む前に。

**2.両方のカバレッジ ステップの `DB_*` 環境変数:**

```yaml
- name: Run backend unit coverage
  id: backend_unit_coverage
  continue-on-error: true
  run: npm run coverage:unit
  working-directory: backend
  env:
    DB_HOST: 127.0.0.1
    DB_PORT: 3306
    DB_DATABASE: TDDTodoAppDB
    DB_USERNAME: root
    DB_PASSWORD: ""

- name: Run backend integration coverage
  id: backend_integration_coverage
  continue-on-error: true
  run: npm run coverage:integration
  working-directory: backend
  env:
    DB_HOST: 127.0.0.1
    DB_PORT: 3306
    DB_DATABASE: TDDTodoAppDB
    DB_USERNAME: root
    DB_PASSWORD: ""
```

一部の単体テスト パスは、ユニット カバレッジ ステップでも DB 認証情報を受け取ります。
インポート時に接続構成を読み取る mysql2 初期化コードを間接的に実行します。
時間。これらの環境変数が存在しない場合、テストが実行される前にプロセスが失敗する可能性があります。

環境変数は、ジョブ レベルではなく各ステップにスコープされます。ステップスコープ内に保持する
CI のみの資格情報であっても、最小特権の原則に従います。
ここには本当の秘密の価値はありません。

**3.統合テスト前の `Build backend` ステップと `Run database migrations` ステップ:**

```yaml
- name: Build backend
  run: npm run build
  working-directory: backend

- name: Run database migrations
  run: node dist/infrastructure/migrate.mjs
  working-directory: backend
  env:
    DB_HOST: 127.0.0.1
    DB_PORT: 3306
    DB_DATABASE: TDDTodoAppDB
    DB_USERNAME: root
    DB_PASSWORD: ""
```

移行スクリプトは、esbuild によって `src/infrastructure/migrate.ts` から
`dist/infrastructure/migrate.mjs`。 `npm ci` を単独で実行すると、このファイルは生成されません。
ビルド ステップは移行ステップよりも前に行う必要があり、移行ステップは移行ステップよりも前に行う必要があります。
統合テストステップ — テストを実行する前にスキーマが存在している必要があります。

### 結果として得られるステップ順序

```
Checkout
→ Setup Node.js
→ Install dependencies (frontend / backend)
→ Run frontend coverage              (continue-on-error)
→ Run backend unit coverage          (continue-on-error + DB env)   ★ env added
→ Build backend                                                       ★ new
→ Run database migrations            (DB env)                         ★ new
→ Run backend integration coverage   (continue-on-error + DB env)   ★ env added
→ Generate combined coverage dashboard
→ Upload artifacts
→ Fail when coverage failed
```

---

## 修正 2 — 実稼働移行で `deploy.yml` に `.env` がロードされない

### 症状

`main` へのプッシュ後、`Run database migrations` ステップが VPS に SSH 接続され、
実行されましたが、認証が失敗したか、間違ったデータベースをターゲットにしました。 SSHセッション
それ自体は成功しました。問題はサーバー上で実行されている Node.js プロセス内にありました。

### 根本的な原因

デプロイ ワークフローでは次のように移行が実行されました。

```yaml
# Before
- name: Run database migrations
  env:
    SSH_USER: ${{ secrets.SAKURA_USER }}
    SSH_HOST: ${{ secrets.SAKURA_HOST }}
  run: |
    ssh "$SSH_USER@$SSH_HOST" \
      "cd ${{ env.DEPLOY_PATH }}/backend && node dist/infrastructure/migrate.mjs"
```

VPS では、データベースの認証情報は次の場所にのみ存在します。
`/var/www/tdd-todo-app/backend/.env`。このファイルはリポジトリにコミットされることはありません
ワークフローによって再同期されることはありません。これは 1 回限りのサーバー セットアップ アーティファクトです。

`node dist/infrastructure/migrate.mjs` が実行されると、移行スクリプトは DB を読み取ります。
`process.env` からの接続構成。 `.env` ファイルは何もロードされなかったので、
`process.env.DB_HOST`、`process.env.DB_PASSWORD` などはすべて `undefined` でした。 mysql2
ドライバーがコンパイル済みのデフォルトに戻ったため、
本番環境の MySQL インスタンス。

このコントラストにより、バグが明確になります。

|プロセス | `.env` のロード方法 |
|---|---|
|アプリケーションサーバー (pm2) | `ecosystem.config.cjs`の`node_args: '--env-file=/var/www/tdd-todo-app/backend/.env'` |
|移行スクリプト (修正前) |何もありません — 環境変数が未定義、ドライバーはデフォルトを使用します。

サーバー プロセスには最初から `--env-file` が与えられていました (前の記事で説明しています)。
パイプラインのデプロイに関する記事)。移行ステップには同等のものはありませんでした。

### 修正 (`3087fd6` をコミット)

`node` 呼び出しに追加された 1 つのフラグ:

```yaml
# After
- name: Run database migrations
  env:
    SSH_USER: ${{ secrets.SAKURA_USER }}
    SSH_HOST: ${{ secrets.SAKURA_HOST }}
  run: |
    ssh "$SSH_USER@$SSH_HOST" \
      "cd ${{ env.DEPLOY_PATH }}/backend && node --env-file .env dist/infrastructure/migrate.mjs"
```

`--env-file` は、Node.js 20 以降の組み込みフラグで、キーと値のペアを
スクリプトの実行を開始する前に、指定されたファイルを `process.env` にコピーします。追加なし
パッケージ (`dotenv`、`dotenv-cli` など) が必要です。Node.js がネイティブに処理します。

パス `.env` は、によって設定された作業ディレクトリに対する相対パスです。
`cd ${{ env.DEPLOY_PATH }}/backend` なので、次のように解決されます。
`/var/www/tdd-todo-app/backend/.env` — pm2 プロセスと同じ認証情報ファイル
すでにロードされています。

移行スクリプト自体には変更は必要ありません。

### `test-coverage.yml` のようにステップレベルの `env:` を使用しないのはなぜですか?

カバレッジ ワークフローでは、アクション ランナー自体が Node.js プロセスをフォークするため、
GitHub Actions のステップレベルの `env:` は、そのプロセスに変数を直接挿入します。
環境 — 完璧に動作します。

デプロイワークフローでは、Node.js プロセスがリモート VPS の子として実行されます。
SSH セッション。アクション ランナーのステップ レベルの `env:` は SSH 境界を越えません。
資格情報はサーバー上にすでに存在している必要があります。 `--env-file` は正しいメカニズムです
ランナーが直接生成しないプロセスにそれらをロードします。

---

## まとめ

|バグ |ワークフロー |根本原因 |修正 |
|---|---|---|---|
|統合テストは常に失敗します。 `test-coverage.yml` | MySQL コンテナはありません。 DB 環境変数はありません。コンパイルされた移行スクリプトはありません。スキーマがありません |ヘルスチェック付きの `services.mysql` を追加します。カバレッジ ステップで `DB_*` 環境変数を設定します。 `Build backend` + `Run database migrations` ステップを追加 |
|本番移行で間違った資格情報が使用される | `deploy.yml` | `node migrate.mjs` は、サーバー側の `.env` をロードせずに VPS 上で実行されました。 `--env-file .env` を `node` 呼び出しに追加します。

どちらのバグも同じ失敗クラス、つまりデータベース接続に依存するステップを共有しています。
資格情報とインフラストラクチャがすでに範囲内にあることを前提としていました。そうではありませんでした。

**データベースに依存するステップをワークフローに追加する場合のチェックリスト:**

1. **ランナーローカル実行の場合:** `services:` コンテナーは定義されていますか? DB の準備ができるまでステップを遅らせる `--health-cmd` はありますか?
2. **接続構成:** `DB_*` 環境変数はステップ レベルで明示的に設定されていますか?
3. **ビルド アーティファクト:** 移行スクリプトは呼び出される前にコンパイルされていますか?
4. **リモート実行 (SSH) の場合:** リモート プロセスには、サーバー側ファイル (`node --env-file .env` など) から認証情報をロードするメカニズムがありますか?

それぞれのケースに対する修正は限定的です。特定の実行コンテキストでできることを提供します。
接続されていない周囲の構成を想定するのではなく、実際にアクセスします。
