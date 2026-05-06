# GitHub Actions の test-coverage ワークフローに MySQL サービスとマイグレーションを追加して CI を修正した

## 対象読者

- GitHub Actions で統合テストを CI に組み込もうとしているバックエンドエンジニア
- Vitest の統合テストが CI 上でデータベース接続エラーで落ちて困っている人
- esbuild でビルドした成果物をワークフローの中間ステップとして使う構成を検討している人

---

## 問題の背景

`test-coverage.yml` ワークフローの `coverage` ジョブは、毎回 `exit 1` で終了していた。

原因は単純だった。`Run backend integration coverage`（`npm run coverage:integration`）ステップは MySQL に接続して統合テストを実行するが、**ジョブに MySQL サービスコンテナが一切定義されておらず、DB 接続情報の環境変数も渡されていなかった**。

```
coverage:integration → MySQL へ接続しようとする
→ 接続先が存在しない → テスト実行失敗
→ Fail when coverage failed ステップが exit 1 を実行
→ CI が常に赤くなる
```

この状態では、push・PR のたびに CI が真っ赤になる。カバレッジの閾値チェックとしての意味を果たしていなかった。

---

## 原因の整理

| 欠けていたもの | 影響 |
|---|---|
| `services.mysql` ブロック | 統合テストが接続する MySQL コンテナが存在しない |
| DB 接続情報の `env` | `DB_HOST` 等が未定義のまま mysql2 が接続しようとして失敗 |
| `Build backend` ステップ | マイグレーションスクリプト（`dist/infrastructure/migrate.mjs`）のビルド成果物が存在しない |
| `Run database migrations` ステップ | スキーマが存在しない状態で統合テストを実行していた |

---

## 修正内容

コミット `fix: add MySQL service and migration step to test-coverage workflow`（SHA: `98a7490`）で、`test-coverage.yml` に 41 行を追加した。

### 1. `services:` ブロックの追加

`coverage` ジョブに MySQL 8.0 のサービスコンテナを定義した。

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

**ポイント:**

- パスワードなし（`MYSQL_ALLOW_EMPTY_PASSWORD: "yes"`）で CI 専用として割り切った構成。
- `--health-cmd` でヘルスチェックを設定し、MySQL が完全に起動してから後続ステップが実行されるようにしている。これがないと、コンテナ起動直後の接続拒否でテストが落ちることがある。
- ホストからは `127.0.0.1:3306` で接続する。GitHub Actions のサービスコンテナはジョブのランナーと同じネットワーク名前空間を共有するため、`localhost` / `127.0.0.1` でアクセスできる。

### 2. DB 接続情報の `env` をステップに追加

`Run backend unit coverage` と `Run backend integration coverage` の両ステップに環境変数を追加した。

```yaml
env:
  DB_HOST: 127.0.0.1
  DB_PORT: 3306
  DB_DATABASE: TDDTodoAppDB
  DB_USERNAME: root
  DB_PASSWORD: ""
```

バックエンドは mysql2 のコネクションプールがこれらの環境変数を読んで接続先を決定する。ローカル開発では `.env` ファイルから読み込まれるが、CI では環境変数を直接渡す必要がある。

ユニットカバレッジステップにも同じ `env` を追加しているのは、ユニットテストの中に mysql2 の初期化を間接的に呼ぶパスが存在するためで、接続情報がなければ起動時にエラーが発生する可能性がある。

### 3. `Build backend` ステップの追加

```yaml
- name: Build backend
  run: npm run build
  working-directory: backend
```

マイグレーションスクリプトは esbuild でトランスパイル・バンドルされた `dist/infrastructure/migrate.mjs` から実行される。`npm ci` だけではビルド成果物が存在しないため、マイグレーションステップの直前にビルドを実行するよう追加した。

### 4. `Run database migrations` ステップの追加

```yaml
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

統合テストが実行される前にスキーマを作成するためのステップ。テーブルが存在しない状態でテストを走らせると、INSERT や SELECT が即座に失敗する。

---

## 修正後のステップ順序

```
Checkout
→ Setup Node.js
→ Install dependencies (frontend / backend)
→ Run frontend coverage              ← continue-on-error: true
→ Run backend unit coverage          ← continue-on-error: true  ★ env 追加
→ Build backend                      ← ★ 新規追加（migrate.mjs のビルド）
→ Run database migrations            ← ★ 新規追加（スキーマ作成）
→ Run backend integration coverage   ← continue-on-error: true  ★ env 追加
→ Generate combined coverage dashboard
→ Upload artifacts
→ Fail when coverage failed          ← すべて成功すれば通過
```

`continue-on-error: true` を使って各カバレッジステップの結果を独立して収集し、最後の `Fail when coverage failed` ステップで一括して判定する設計はそのまま維持されている。

---

## 注意点

### ヘルスチェックの重要性

`options` の `--health-cmd` を省略すると、MySQL コンテナが起動中（accepting connections の前）にマイグレーションや接続テストが走ってしまい、`connection refused` で失敗する。リトライ間隔（`--health-interval=10s`）とタイムアウト（`--health-timeout=5s`）も適切に設定しておくこと。

### ビルドとマイグレーションの順序依存

`Build backend` → `Run database migrations` の順序は厳守する必要がある。ビルドなしでマイグレーションを実行しようとすると「`dist/infrastructure/migrate.mjs` が見つからない」というエラーになる。GitHub Actions のステップは上から順に実行されるため、順序をコード上で明示的に管理できるのは利点である。

### 環境変数のスコープ

環境変数はステップレベルで設定されており、ジョブ全体には公開されていない。`DB_PASSWORD: ""` のような空文字を含む機密情報でも、ジョブレベルに広げるよりもステップ単位に限定するほうが最小権限の原則に近い。

---

## まとめ

| 変更前 | 変更後 |
|---|---|
| MySQL サービスコンテナなし | mysql:8.0 サービスコンテナあり（ヘルスチェック付き） |
| DB 接続情報なし | `DB_HOST` 等を各ステップの `env` で設定 |
| ビルドステップなし | `npm run build` でマイグレーション用成果物を生成 |
| マイグレーションなし | `node dist/infrastructure/migrate.mjs` でスキーマを事前作成 |
| CI が常に赤 | 統合カバレッジが正常に実行されるようになった |

統合テストを CI に組み込む際、テスト自体の実装と「CI がテスト実行に必要なインフラを準備できているか」は別の問題として意識する必要がある。今回の修正は後者が抜け落ちていたケースで、チェックリストとして「サービスコンテナ・接続情報・スキーマ」の三点を確認する習慣が有効だとわかる事例だった。
