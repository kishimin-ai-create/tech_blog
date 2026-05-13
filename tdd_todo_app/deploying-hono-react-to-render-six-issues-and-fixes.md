# Hono + React アプリをデプロイしてレンダリングする: 遭遇した 6 つの問題とその解決方法

**対象読者**: Node.js + React フルスタック アプリを初めて Render にデプロイし、予想される摩擦点について実践的に説明したいエンジニア。

**スタック**: React + Vite (静的サイト)、Hono + Node.js (Web サービス)、Filess.io 上の MySQL、レンダー ブループリント (`render.yaml`)

---

## 背景

TDD Todo アプリは、MySQL データベースを使用してローカルで実行されていました。このセッションの目標は、ブループリント機能 (単一の信頼できるソースからすべてのサービスをプロビジョニングおよび自動デプロイするリポジトリにコミットされた `render.yaml` ファイル) を使用して、[Render](https://render.com/) 上のライブ環境にプロモートすることでした。

単純な構成プッシュのように見えた作業が、6 か所にわたるデバッグ ツアーに変わりました。それぞれの問題は単独では小規模でしたが、これらを総合すると、ブランチ管理、プランの選択、ビルド時の依存関係、クロスオリジン ポリシー、データベースの初期化、ORM の日付タイプの不一致など、最初の Render デプロイメントで発生する可能性のある問題のほぼすべてのカテゴリがカバーされます。

---

## 問題 1 — `render.yaml` は機能ブランチにのみ存在しました

### 症状

レンダー ブループリントは、デフォルト ブランチ (`main`) から `render.yaml` を読み取ります。ファイルは `frontend` ブランチ上に作成され、マージされていなかったため、Render はそれを見つけることができませんでした。

### Fix

プル リクエストを介して、`frontend` ブランチを `main` にマージします。ファイルが `main` に到達すると、Render はそれを読み取り、両方のサービスを自動的にプロビジョニングできます。

### 取り除く

`render.yaml` は、レンダー ウォッチのブランチ上に存在する必要があります。ブループリントを作成する前に、リポジトリのデフォルト ブランチを確認してください。

---

## 問題 2 — Render は有料プランを要求しました

### 症状

ブループリントの検出後、Render はプロビジョニングをブロックし、バックエンド Web サービスの課金アップグレードを要求しました。

### 根本的な原因

`render.yaml` で `plan` が省略された場合、レンダリングはデフォルトで有料インスタンス タイプになります。

### Fix

`plan: free` をバックエンド サービス定義に明示的に追加します。

```yaml
# render.yaml
- type: web
  name: tdd-todo-app-backend
  env: node
  plan: free          # ← required to stay on the free tier
  buildCommand: cd backend && npm ci --include=dev && npm run build && node dist/infrastructure/migrate.mjs
  startCommand: cd backend && npm start
```

### 取り除く

Render のブループリント仕様では、無料利用枠を想定していません。それが意図されている場合は、常に `plan: free` を宣言してください。

---

## 問題 3 — ビルド中に `esbuild` が見つからない

### 症状

ビルド ステップが次のようなエラーで失敗しました。

```
sh: esbuild: not found
```

バックエンドは、`esbuild` を使用して、`src/server.ts` と `src/infrastructure/migrate.ts` を `dist/` にバンドルします。

```json
"build:server": "esbuild src/server.ts --bundle --platform=node --packages=external --outfile=dist/server.mjs --format=esm"
```

### 根本的な原因

レンダリングは、`buildCommand` を実行する前に `NODE_ENV=production` を設定します。 `NODE_ENV=production`、`npm ci` の場合、デフォルトで `devDependencies` がスキップされ、`esbuild` は `devDependencies` 内に存在します。結果: バンドラーはビルド時に存在しません。

### Fix

開発依存関係がビルドフェーズ中にインストールされるように、`--include=dev` を `npm ci` に渡します。

```yaml
buildCommand: cd backend && npm ci --include=dev && npm run build && node dist/infrastructure/migrate.mjs
```

### 取り除く

`npm ci` は `NODE_ENV` を尊重します。 `devDependencies` に存在するビルド ツール (バンドラー、型チェッカー、テスト ランナー) は、`NODE_ENV=production` がプラットフォームによって設定されるときに明示的に含める必要があります。あるいは、必要なビルド ツールのみを `dependencies` に移動しますが、`--include=dev` の方がビルド ステップのより単純なグローバル ソリューションです。

---

## 問題 4 — CORS エラーによりフロントエンドがブロックされた

### 症状

両方のサービスが実行されると、ブラウザ コンソールに次のように表示されます。

```
Access to fetch at 'https://tdd-todo-app-backend.onrender.com/api/v1/apps'
from origin 'https://tdd-todo-app-frontend.onrender.com' has been blocked
by CORS policy: No 'Access-Control-Allow-Origin' header is present.
```

フロントエンドとバックエンドは異なるサブドメイン上にあるため、すべての API 呼び出しはクロスオリジン リクエストとして扱われました。 Hono アプリには CORS ミドルウェアが構成されていませんでした。

### Fix

`hono/cors` をグローバル ミドルウェアとして追加し、環境変数を通じて許可されたオリジンを駆動します。

```typescript
// backend/src/infrastructure/hono-app.ts
import { cors } from 'hono/cors';

export function createHonoApp(dependencies: HonoAppDependencies): Hono {
  const app = new Hono();

  app.use(
    '*',
    cors({
      origin: process.env.CORS_ORIGIN ?? '*',
      allowMethods: ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS'],
      allowHeaders: ['Content-Type'],
    }),
  );
  // ... routes
}
```

レンダリング ダッシュボードで、バックエンド サービスの `CORS_ORIGIN` 環境変数を `https://tdd-todo-app-frontend.onrender.com` に設定します。許可されたオリジンを環境変数に保持すると、デプロイメント固有の URL をソース コードにハードコーディングすることが回避され、`*` をローカル フォールバックとして保持しながら、運用環境でアクセスを適切に制限することが容易になります。

### 取り除く

フロントエンドとバックエンドが異なるオリジンにある場合は常に (同じプラットフォームの異なるサブドメインであっても)、バックエンドは明示的に CORS ヘッダーを発行する必要があります。許可されたオリジンに環境変数を使用すると、デプロイメント環境ごとに異なる値を処理できるクリーンな方法になります。

---

## 問題 5 — DB テーブルが存在しないため、起動時にアプリがクラッシュする

### 症状

バックエンドは正常に開始されましたが、最初の API リクエストで 500 エラーが返されました。レンダリング ログには、`App` テーブルと `Todo` テーブルが欠落していることを示す MySQL エラーが表示されました。

### 根本的な原因

移行スクリプト (`dist/infrastructure/migrate.mjs`) は、Render MySQL インスタンスで実行されませんでした。ローカルでは、移行は `npm run migrate` を使用して手動で実行されました。クラウド展開パイプラインには同等のステップがありませんでした。

### Fix

移行スクリプトを `buildCommand` に追加して、バンドルのコンパイル後、サービスの開始前に 1 回実行されるようにします。

```yaml
buildCommand: >
  cd backend &&
  npm ci --include=dev &&
  npm run build &&
  node dist/infrastructure/migrate.mjs
```

移行スクリプトはべき等です。`_migrations` テーブル内の適用されたファイルを追跡し、すでに実行されているファイルはスキップします。したがって、デプロイごとに実行しても安全です。

```typescript
// From migrate.ts — idempotency guard
const [rows] = await connection.execute<MigrationRow[]>(
  'SELECT name FROM _migrations WHERE name = ?',
  [file],
);
if (rows.length > 0) {
  console.log(`  skip   ${file}`);
  continue;
}
```

### 取り除く

移行ツールが冪等である場合、デプロイ パイプラインの一部として実行するのが最も安全なアプローチです。これにより、手動の手順や個別のレンダリング ジョブを必要とせずに、スキーマが常に最新の状態に保たれます。

---

## 問題 6 — MySQL が ISO 8601 日時文字列を拒否する

### 症状

アプリを作成すると HTTP 500 が返されました。レンダリング ログには次のことが示されました。

```
ER_TRUNCATED_WRONG_VALUE: Incorrect datetime value: '2026-05-06T12:01:53.756Z' for column 'createdAt' at row 1
```

### 根本的な原因

アプリケーションのドメイン層は、タイムスタンプを ISO 8601 文字列 (例: `"2026-05-06T12:01:53.756Z"`) として保存します。これらの文字列がバインド パラメータとして `mysql2` に直接渡された場合、MySQL の `DATETIME` 列は `T` セパレータと `Z` サフィックスを拒否しました。`YYYY-MM-DD HH:MM:SS` 形式が想定されます。

`mysql2` 自体は、JavaScript `Date` オブジェクトを正しい形式にシリアル化できますが、リポジトリは生の ISO 文字列を渡していました。

```typescript
// Before — passes ISO string directly
[app.id, app.name, app.createdAt, app.updatedAt, app.deletedAt]
```

### Fix

各 ISO 文字列をクエリに渡す前に `new Date()` でラップします。次に、`mysql2` は、`Date` オブジェクトを MySQL が受け入れる形式にシリアル化します。

```typescript
// After — mysql-app-repository.ts
[
  app.id,
  app.name,
  new Date(app.createdAt),
  new Date(app.updatedAt),
  app.deletedAt ? new Date(app.deletedAt) : null,
]
```

同じ修正が `mysql-todo-repository.ts` に適用されました。

```typescript
// After — mysql-todo-repository.ts
[
  todo.id,
  todo.appId,
  todo.title,
  todo.completed,
  new Date(todo.createdAt),
  new Date(todo.updatedAt),
  todo.deletedAt ? new Date(todo.deletedAt) : null,
]
```

`null` は `null` のままでなければならないことに注意してください。これを `new Date()` でラップすると、`Invalid Date` が生成されます。

### 取り除く

MySQL の `DATETIME` 型は ISO 8601 形式を受け付けません。ドメインモデルで日付を文字列として保持している場合は、`mysql2` に渡す前に `new Date()` で変換してください。ドライバは `Date` オブジェクトから正しくシリアライズを行い、任意の文字列形式を解釈しようとはしません。

---

## 最終 `render.yaml`

6 つの修正をすべて行うと、ブループリントの定義は次のようになります。

```yaml
services:
  # Frontend (Static Site)
  - type: web
    name: tdd-todo-app-frontend
    env: static
    buildCommand: cd frontend && npm ci && npm run build
    staticPublishPath: frontend/dist
    routes:
      - type: rewrite
        source: /*
        destination: /index.html

  # Backend (Web Service)
  - type: web
    name: tdd-todo-app-backend
    env: node
    plan: free
    buildCommand: cd backend && npm ci --include=dev && npm run build && node dist/infrastructure/migrate.mjs
    startCommand: cd backend && npm start
    envVars:
      - key: NODE_ENV
        value: production
      - key: DATABASE_URL
        sync: false
```

`DATABASE_URL` は `sync: false` とマークされているため、その値はリポジトリにコミットされることはありません。値はレンダー ダッシュボードで一度設定され、そこに安全に保持されます。

---

## まとめ

| # |問題 |根本原因 |修正 |
|---|---------|------------|-----|
| 1 |レンダリングで `render.yaml` が見つかりませんでした |ファイルは `main` ではなく機能ブランチ上にありました。 `main` にマージ |
| 2 |レンダリングが有料プランを要求しました | `plan` キーがありませんでした。デフォルトは有料です | `plan: free`を追加 |
| 3 | `esbuild` がビルド時に見つかりません。 `NODE_ENV=production` | の場合、`npm ci` は `devDependencies` をスキップします。 `npm ci --include=dev` を使用する |
| 4 | CORS はすべての API リクエストをブロックしました |バックエンドに CORS ヘッダーがない | `hono/cors` ミドルウェアを追加し、`CORS_ORIGIN` env var | 経由で設定します。
| 5 |最初のデプロイ時にはテーブルが存在しませんでした。移行はクラウド パイプラインで実行されたことはありません。 `node dist/infrastructure/migrate.mjs` を `buildCommand` に追加 |
| 6 | MySQL が日時値を拒否しました。 ISO 8601 文字列は有効な MySQL `DATETIME` 入力ではありません。文字列を `mysql2` に渡す前に `new Date()` でラップします。

これらの問題はそれぞれ、ほとんどの最初の Render デプロイメントで発生するほど一般的なものです。同様のスタックをデプロイする場合は、プッシュする前にこれらの 6 つのポイントを確認すると、再デプロイのサイクルを数回節約できる可能性があります。
