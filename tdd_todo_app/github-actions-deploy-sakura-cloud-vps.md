# GitHub ActionsによるさくらのクラウドVPSへの自動デプロイ

**タイプ:** 開発ブログ / リリースノート
**日付:** 2026-05-04
**対象読者:** Linux VPS 上で Node.js + Vite アプリのプッシュ デプロイ パイプラインを設定したいエンジニア、またはこのプロジェクトの実稼働デプロイメントがどのように機能するかを追跡している人。

---

## 概要

このセッションでは、さくらのクラウド VPS でホストされている TDD Todo アプリ (Hono/Node.js API と MySQL データベースによってサポートされる React/Vite フロントエンド) の完全に自動化されたデプロイメント パイプラインを接続しました。

結果: `main` へのプッシュごとに、スタックの両端が構築され、テスト スイートがゲートとして実行され、アーティファクトがサーバーに rsync され、データベースがバックアップされ、保留中の移行が実行され、pm2 経由で Node.js プロセスがリロードされます。これらはすべて手動でサーバーに触れることなく行われます。

5 つのコミットが関係していました。

|シャ |説明 |
|---|---|
| `22f3dc8` |初期展開ワークフロー + pm2 エコシステム構成 + esbuild スクリプト |
| `5c30648` |セキュリティ強化: 固定されたホストキー、テストゲート、秘密の分離、DB バックアップ |
| `efad45e` | `eslint-plugin-unused-imports` を `devDependencies` に移動 |
| `98f089d` | pm2 設定とドキュメント パスの結合で `NODE_ENV=production` を設定します。
| `9bf6795` |コード レビュー ドキュメントに処理と返信を追加する |

---

## 何がどのように展開されるのか

### フロントエンド — Vite → rsync → Nginx

```
frontend/src  →  npm run build  →  frontend/dist/
                                        ↓
                              rsync --delete  →  /var/www/tdd-todo-app/frontend/
                                        ↓
                                   served by Nginx
```

`rsync -avz --delete` は、デプロイのたびに、以前のビルドからの古いファイルが確実に削除されるようにします。

### バックエンド — TypeScript → esbuild → rsync → pm2

2 つの esbuild コマンドは、TypeScript ソースを ESM `.mjs` バンドルにコンパイルします。

```jsonc
// backend/package.json
"build:server":   "esbuild src/server.ts   --bundle --platform=node --packages=external --outfile=dist/server.mjs   --format=esm",
"build:migrate":  "esbuild src/infrastructure/migrate.ts --bundle --platform=node --packages=external --outfile=dist/infrastructure/migrate.mjs --format=esm"
```

`--packages=external` は重要なフラグです。npm の依存関係は出力にバンドルされません**。代わりに、`package.json` と `package-lock.json` は `dist/` ディレクトリと一緒に rsync され、`npm ci --omit=dev` は運用依存関係のみをサーバーにインストールします。これにより、通常の `node_modules` 解決パスを維持しながら、バンドルを小さく保ちます。

### pm2プロセス管理

`backend/ecosystem.config.cjs` は、デプロイごとにサーバーに rsync される pm2 構成ファイルです。

```js
module.exports = {
  apps: [{
    name: 'tdd-todo-app',
    script: './dist/server.mjs',
    cwd: '/var/www/tdd-todo-app/backend',
    node_args: '--env-file=/var/www/tdd-todo-app/backend/.env',
    instances: 1,
    autorestart: true,
    watch: false,
    max_memory_restart: '300M',
    env: { NODE_ENV: 'production' },
  }],
};
```

再起動ステップでは冪等ロジックを使用します。プロセスがすでに存在する場合は `pm2 reload`、最初のデプロイ時は `pm2 start` です。

```yaml
- name: Restart backend service
  run: |
    ssh "$SSH_USER@$SSH_HOST" << 'EOF'
      set -e
      if pm2 describe tdd-todo-app > /dev/null 2>&1; then
        pm2 reload tdd-todo-app --update-env
      else
        pm2 start /var/www/tdd-todo-app/backend/ecosystem.config.cjs
      fi
      pm2 save
    EOF
```

`pm2 reload` は、ダウンタイムなしでローリング再起動を実行します (古いプロセスが終了する前に接続がハンドオフされます)。 `--update-env` は、エコシステム設定からの新しい env エントリが確実に有効になるようにします。

### データベースの移行

移行が実行される前に、タイムスタンプ付きの `mysqldump` スナップショットが取得されます。

```yaml
- name: Backup database before migration
  run: |
    ssh "$SSH_USER@$SSH_HOST" \
      "mysqldump --defaults-file=${{ env.DEPLOY_PATH }}/backend/.my.cnf \
        TDDTodoAppDB > /var/backups/tdd-todo-app-$(date +%Y%m%d%H%M%S).sql"

- name: Run database migrations
  run: |
    ssh "$SSH_USER@$SSH_HOST" \
      "cd ${{ env.DEPLOY_PATH }}/backend && node dist/infrastructure/migrate.mjs"
```

`.my.cnf` ファイル (1 回限りのサーバー セットアップ手順) には MySQL 資格情報が保持されるため、プロセス リストに単純な CLI 引数として表示されることはありません。

---

## セキュリティの強化 — コードレビューの調査結果と修正

最初のコミット (`22f3dc8`) のコード レビューにより、2 つの優先度レベルにわたって 6 つの問題が特定されました。マージ前にすべて解決されました。

### P1: ssh-keyscan は TOFU の脆弱性です

**前に：**
```yaml
- name: Add server to known_hosts
  run: ssh-keyscan -H ${{ secrets.SAKURA_HOST }} >> ~/.ssh/known_hosts
```

`ssh-keyscan` は、スキャン時にサーバーが返すキーを信頼します。その時点で DNS または BGP ルートをハイジャックできる攻撃者は、ジョブ内のすべての `ssh` および `rsync` 呼び出しに対してキーを信頼されます。

**後:** サーバーの公開キーは (信頼できるネットワークから取得して) GitHub Secret `SAKURA_KNOWN_HOST` に保存され、直接エコーされます。

```yaml
- name: Add server to known_hosts
  env:
    SAKURA_KNOWN_HOST: ${{ secrets.SAKURA_KNOWN_HOST }}
  run: |
    mkdir -p ~/.ssh
    echo "$SAKURA_KNOWN_HOST" >> ~/.ssh/known_hosts
```

これにより、予想されるフィンガープリントが固定され、展開ごとに TOFU ウィンドウが排除されます。

### P1: テストゲートがない - 壊れたコードが本番環境に到達する可能性がある

最初のワークフローは、`npm run build` から SSH デプロイメントに直接進みました。テストを中断したコミットは引き続きデプロイされます。

**修正:** 両側のインストールとビルドの間にインライン テスト ステップが追加されました。

```yaml
- name: Run frontend tests
  working-directory: frontend
  run: npm test

- name: Run backend unit tests
  working-directory: backend
  run: npm run test:unit
```

統合テストはここでは意図的に除外されています。統合テストには、アクション ランナーでは使用できないライブ MySQL インスタンスが必要です。単体テストは、展開ワークフローの適切なゲートです。クロスワークフロー `needs:` (別の `backend.yaml` CI ジョブに応じて) は、異なるワークフロー ファイル内のジョブの GitHub Actions では実行できないため、テストはインラインで行われます。

### P1: 事前のデータベース バックアップなしで移行が実行されました

移行には、元に戻せない DDL が含まれます。バックアップがなければ、破壊的な `ALTER` または `DROP` により、回復パスがなくなり、永久的なデータ損失が発生する可能性があります。

**修正:** 上記の `mysqldump` バックアップ ステップが、移行ステップの直前に追加されました。

### P2: シェル文字列に直接挿入されたシークレット

`ssh ${{ secrets.SAKURA_USER }}@${{ secrets.SAKURA_HOST }}` のようなパターンは、シークレット値をシェル コマンドに直接展開します。シークレット値にシェル メタ文字（`;`、バッククォート、`$(...)`）が含まれていた場合、それらはアクション ランナーで実行されます。

**修正:** すべての ssh/rsync ステップは、ステップレベルの `env:` ブロックを介してシークレットを公開し、それらをシェル変数として参照するようになりました。

```yaml
- name: Deploy frontend
  env:
    SSH_USER: ${{ secrets.SAKURA_USER }}
    SSH_HOST: ${{ secrets.SAKURA_HOST }}
  run: |
    rsync -avz --delete frontend/dist/ "$SSH_USER@$SSH_HOST:${{ env.DEPLOY_PATH }}/frontend/"
```

### P2: `eslint-plugin-unused-imports` は `dependencies` 内にありました

この lint 専用パッケージはランタイムでの使用はありませんでしたが、`dependencies` の下にリストされていたため、`npm ci --omit=dev` はすべての実稼働デプロイにこのパッケージ (およびその推移的ツリー) をインストールし、制約のある VPS でディスク容量とインストール時間を無駄にしていました。

**修正:** `devDependencies` に移動されました。実稼働環境でのインストールがよりスリムになりました。

### P2: `NODE_ENV` が `production` であることは決して保証されませんでした

オペレーターが `NODE_ENV=production` を追加せずに `.env.example` を `.env` にコピーした場合、サーバー プロセスは `NODE_ENV` が未定義の状態で実行されます。 `NODE_ENV === 'production'` 上の Hono ゲートのプロダクションセーフなエラー処理などのライブラリ。

**修正:** pm2 アプリ構成に `env: { NODE_ENV: 'production' }` を追加しました。 pm2 `env` ブロックは `--env-file` よりも優先されるため、オペレーターが `.env` に何を書いたかに関係なく、運用モードが保証されます。

### P3: `ecosystem.config.cjs` と `deploy.yml` 間のパス結合

`ecosystem.config.cjs` は、`cwd` と `node_args` の両方で `/var/www/tdd-todo-app/backend` をハードコードします。ワークフローは、`env.DEPLOY_PATH` と同じルートを定義します。一方を変更せずに他方を変更すると、pm2 はパスが見つからないエラーで失敗します。

**修正:** 両方の行に目立つ `// IMPORTANT: must match DEPLOY_PATH in .github/workflows/deploy.yml` コメントを追加し、ファイル レベルの JSDoc に `PATH COUPLING` ブロックを追加しました。テンプレートの生成が検討されましたが、延期されました。

---

## 必要な GitHub シークレット

|秘密 |値 |
|---|---|
| `SAKURA_SSH_PRIVATE_KEY` | SSH 秘密鍵 (Ed25519 または RSA) |
| `SAKURA_HOST` |サーバーのホスト名または IP |
| `SAKURA_USER` | SSH ユーザー名 |
| `SAKURA_KNOWN_HOST` |固定されたサーバーの公開キー - 信頼できるネットワークから `ssh-keyscan -H <server>` を 1 回実行します。 |

すべてのシークレットは、GitHub の `production` 環境 (ワークフローでは `environment: production`) にスコープされているため、デプロイの実行中にのみアクセスできます。

---

## ワンタイムサーバーセットアップ

最初のデプロイの前に、VPS には以下が必要です。

1. **Node.js 20+**、npm、および pm2 (`npm install -g pm2`)
2. **MySQL** が実行中でアクセス可能
3. `/var/www/tdd-todo-app/backend/.env` — `.env.example` からコピーし、DB 資格情報を入力します
4. `/var/www/tdd-todo-app/backend/.my.cnf` — `mysqldump` の MySQL 認証情報 (プロセス リスト内のプレーンテキスト パスワードを回避します)
5. `pm2 startup && pm2 save` — pm2 を systemd サービスとして登録し、サーバーの再起動後もプロセスが存続するようにします
6. **Nginx** は、`frontend/` を静的ファイルとして提供し、`/api` をバックエンド ポートにリバース プロキシするように構成されています

---

## 完全なワークフローの概要

```
push to main
  │
  ├── Install frontend deps  →  Run frontend tests  →  Build frontend (Vite)
  ├── Install backend deps   →  Run backend unit tests  →  Build backend (esbuild)
  │
  ├── Setup SSH agent + add pinned known_hosts
  │
  ├── rsync frontend/dist/  →  /var/www/tdd-todo-app/frontend/
  ├── rsync backend/dist/ + migrations/ + package*.json + ecosystem.config.cjs
  ├── npm ci --omit=dev  (on server)
  │
  ├── mysqldump  →  /var/backups/tdd-todo-app-<timestamp>.sql
  ├── node dist/infrastructure/migrate.mjs  (on server)
  │
  └── pm2 reload tdd-todo-app  (or pm2 start on first deploy)
```

---

## まとめ

デプロイ パイプラインは、単一のセッションでゼロから運用環境に安全な状態になりました。最初の実装では順序が正しく (ビルド → デプロイ → 移行 → 再起動)、いくつかの適切なデフォルト (`environment: production`、リモート シェルのヒアドキュメント引用) が使用されました。その後、コード レビューで 4 つのセキュリティ/信頼性のギャップ (TOFU ホスト キーの信頼性、テスト ゲートの欠落、保護されていない移行、シークレット インジェクションのリスク) が見つかりました。これらはすべて、次の 2 つのコミットで修正されました。

最終状態は次のようなパイプラインです。
- 単体テストに失敗したコードをデプロイできない
- 導入時に侵害された DNS/BGP パスによって操作できない
- スキーマが変更されるたびに、タイムスタンプ付きのデータベース スナップショットを保存します。
- シークレット値がシェルコマンド文字列に含まれないようにします。
- オペレーターが指定した `.env` 値に依存するのではなく、pm2 構成を介して `NODE_ENV=production` を無条件に実行します。
