# Cloudflare Pages 自動デプロイを GitHub Actions で設定する — フロントエンドとバックエンドの現実的な判断

## 対象読者

- Vite + React のフロントエンドを Cloudflare Pages にデプロイしたいエンジニア
- Hono バックエンドの Cloudflare Workers 対応を検討中で、現状の制約を把握したいエンジニア
- GitHub Actions で CI/CD パイプラインを整備している方

---

## この記事でカバーすること / しないこと

**カバーする**
- Cloudflare Pages を選んだ理由
- `wrangler.jsonc` と GitHub Actions ワークフローの設定方法
- バックエンドが現状 Workers 未対応な理由と将来方針
- GitHub Secrets の設定手順
- 初回デプロイチェックリスト

**カバーしない**
- Cloudflare D1 / Hyperdrive への実際の移行手順（今後の作業）
- Cloudflare Workers 本番運用のチューニング

---

## 背景

このプロジェクト（TDD Todo App）は **Vite + React + TypeScript** のフロントエンドと **Hono + Node.js + MySQL** のバックエンドで構成されている。

CI は GitHub Actions で整備済みだったが、デプロイ先と自動デプロイの仕組みがなかった。  
コミット `cde58b1`（`feat: add Cloudflare deployment workflows`）で以下の 4 ファイルを追加し、フロントエンドの自動デプロイを有効化した。

| ファイル | 役割 |
|---|---|
| `frontend/wrangler.jsonc` | Cloudflare Pages 用の wrangler 設定 |
| `.github/workflows/deploy-frontend.yml` | main へ push → Pages デプロイ |
| `.github/workflows/deploy-backend.yml` | Workers デプロイのプレースホルダー（TODO） |
| `docs/deployment.md` | デプロイ手順ドキュメント |

---

## なぜ Cloudflare Pages / Workers を選んだか

- **エッジ配信**: 静的アセットをグローバル CDN から配信でき、追加費用なしで高速化できる
- **ゼロコールドスタート（Pages）**: 静的ホスティングなのでサーバーレス特有のコールドスタート問題がない
- **`cloudflare/pages-action` との統合**: GitHub Actions から 1 ステップでデプロイできる公式アクションが存在する
- **将来の Workers 対応**: バックエンドの Hono は Workers でも動作できる設計のため、データ層を移行すれば同じ Cloudflare エコシステムに統一できる

---

## フロントエンドの自動デプロイ設定

### 1. `frontend/wrangler.jsonc`

```jsonc
{
  "name": "tdd-todo-app-frontend",
  "pages_build_output_dir": "./dist",
  "compatibility_date": "2025-01-01"
}
```

**ポイント**
- `name` は Cloudflare ダッシュボードで作成する Pages プロジェクト名と一致させる必要がある
- `pages_build_output_dir` は `npm run build` の出力先（`./dist`）を指定する
- `compatibility_date` は Cloudflare の API バージョンを固定するための日付。新規プロジェクトでは最新安定日を入れておく

### 2. `.github/workflows/deploy-frontend.yml`

```yaml
name: Deploy frontend to Cloudflare Pages

on:
  push:
    branches: [main]
    paths:
      - 'frontend/**'
      - '.github/workflows/**'
  workflow_dispatch:

permissions:
  contents: read
  deployments: write

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
          cache-dependency-path: frontend/package-lock.json

      - name: Install frontend dependencies
        working-directory: frontend
        run: npm ci

      - name: Build frontend
        working-directory: frontend
        run: npm run build

      - name: Deploy to Cloudflare Pages
        uses: cloudflare/pages-action@v1
        with:
          apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          accountId: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
          projectName: tdd-todo-app-frontend
          directory: frontend/dist
          gitHubToken: ${{ secrets.GITHUB_TOKEN }}
          wranglerVersion: '3'
```

**設計上の判断**

| 設定 | 意図 |
|---|---|
| `paths: frontend/**` | フロントエンドに無関係な変更でデプロイが走らないようにする |
| `paths: .github/workflows/**` | ワークフロー自体の変更時も再デプロイしてワークフローが壊れていないか確認できる |
| `permissions: deployments: write` | `cloudflare/pages-action` が GitHub の Deployments API に書き込むために必要 |
| `environment: production` | GitHub Environments と連携し、デプロイ履歴・承認フローを管理できる |
| `wranglerVersion: '3'` | wrangler のバージョンを固定して予期せぬ破壊的変更を防ぐ |

---

## バックエンドが Workers 未対応な理由

### 現状のスタック

```
Hono
 └─ @hono/node-server   ← Node.js ランタイムに依存
      └─ mysql2          ← TCP ソケットで MySQL に直接接続
```

Cloudflare Workers は **V8 Isolate** 上で動作し、Node.js の API（`net`、`fs` など）や TCP ソケットを使えない。  
このため `@hono/node-server` と `mysql2` の TCP 接続は Workers 上では動作しない。

### Workers 対応に必要な作業（今後の方針）

1. **データ層の移行**: `mysql2` の TCP 接続を Workers 対応の手段（Hyperdrive、D1、HTTP アクセス可能なプロキシ）に置き換える
2. **サーバー起動コードの分離**: `@hono/node-server` の起動ロジックと Hono アプリ本体を分割し、Workers が `app` を直接インポートできるようにする
3. **`backend/wrangler.jsonc` と Workers エントリポイントの追加**: `backend/src/worker.ts` のようなエントリポイントを用意する
4. **ワークフローの更新**: `deploy-backend.yml` の TODO を `cloudflare/wrangler-action@v3` の実装に差し替える

### プレースホルダーワークフロー

```yaml
# .github/workflows/deploy-backend.yml（現状）
jobs:
  todo:
    runs-on: ubuntu-latest
    steps:
      - name: Explain current blocker
        run: |
          echo "TODO: Cloudflare Workers deployment is not enabled yet."
          echo "The backend currently uses @hono/node-server plus mysql2-based TCP database access."
          echo "Migrate the runtime and data access to Workers-compatible components before enabling wrangler deploy."
```

ワークフローファイルとして存在させておくことで、将来の実装者が「Workers デプロイはここに書く」とすぐ分かるようにしている。また、現状は `echo` を実行するだけなのでデプロイが壊れることもない。

---

## GitHub Secrets の設定方法

デプロイを動かすには 2 つのシークレットをリポジトリに登録する必要がある。

### 1. `CLOUDFLARE_API_TOKEN`

1. Cloudflare ダッシュボードを開く
2. 右上のユーザーアイコン → **My Profile** → **API Tokens**
3. **Create Token** → テンプレートから「Edit Cloudflare Workers」を選ぶか、カスタムで作成
4. 権限として **Cloudflare Pages: Edit** を付与（後で Workers を有効化するなら **Workers Scripts: Edit** も追加）
5. 生成されたトークン文字列をコピー

### 2. `CLOUDFLARE_ACCOUNT_ID`

1. Cloudflare ダッシュボードのサイドバー右側に表示されている **Account ID** をコピー

### GitHub への登録手順

```
GitHub リポジトリ
 → Settings
 → Secrets and variables
 → Actions
 → New repository secret
```

| Secret 名 | 設定する値 |
|---|---|
| `CLOUDFLARE_API_TOKEN` | Cloudflare で生成した API トークン |
| `CLOUDFLARE_ACCOUNT_ID` | Cloudflare のアカウント ID |

> **注意**: `GITHUB_TOKEN` はリポジトリに自動で提供されるため、手動登録は不要。

---

## 初回デプロイのチェックリスト

GitHub Actions ワークフローを動かす前に、Cloudflare 側で手動でプロジェクトを作成しておく必要がある。  
ワークフローは既存プロジェクトへのデプロイを行うため、プロジェクトが存在しない場合はエラーになる。

```
[ ] 1. Cloudflare ダッシュボード → Workers & Pages → Pages プロジェクトを作成
       プロジェクト名: tdd-todo-app-frontend（wrangler.jsonc の name と一致させる）
[ ] 2. GitHub リポジトリに CLOUDFLARE_API_TOKEN を登録
[ ] 3. GitHub リポジトリに CLOUDFLARE_ACCOUNT_ID を登録
[ ] 4. main ブランチに frontend/ または .github/workflows/ を変更するコミットをプッシュ
[ ] 5. Actions タブで "Deploy frontend to Cloudflare Pages" ワークフローの成功を確認
[ ] 6. Cloudflare ダッシュボードでデプロイされた Pages URL を開いてサイトが表示されることを確認
```

---

## まとめ

| 項目 | 対応状況 |
|---|---|
| フロントエンド（Cloudflare Pages）自動デプロイ | ✅ 完了 |
| バックエンド（Cloudflare Workers）自動デプロイ | 🔲 TODO（プレースホルダーワークフロー設置済み） |

今回の変更で押さえた設計の要点は次の 3 つ。

1. **フロントエンドだけ先行して自動デプロイを有効化**: 技術的制約をブロッカーにせず、可能な部分から段階的に整備した
2. **バックエンドはプレースホルダーワークフローで意図を明示**: 「なぜ未対応か」「何をすれば対応できるか」をコードベースに残しておくことで、将来の実装コストを下げた
3. **パスフィルターで不要なデプロイを防ぐ**: `paths` を絞ることでデプロイ回数と Cloudflare API の使用量を抑えた

バックエンドの Workers 対応は Hyperdrive または D1 への移行が前提となる。移行後は `deploy-backend.yml` の TODO を `cloudflare/wrangler-action@v3` に置き換えるだけで自動デプロイが有効化できる構成になっている。
